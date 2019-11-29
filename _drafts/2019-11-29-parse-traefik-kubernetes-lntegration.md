---
layout: post
title: 'Kubernete Ingress Controller 如何实现配置动态发现(以 traefik 为例的源码分析)'
date: 2019-11-29
author: Calvin Wang
cover: "/assets/img/201911/traefik-architecture.png"
tags: Kubernete Ingress Controller traefik
---

> 最近在调研 Ingress Contoller, 其中有一些问题一直没能解决, 遂决定翻翻源码并记录下.

### #1. 提出问题:
一个请求到达 pod 的路径可以简单理解为:
![normal request](/assets/img/201911/WX20191129-112923.png)
其中默认情况下: 
kube-proxy 会将收到的请求随机分配到一个健康的 Pod 上去. 在一定程度上就在承担着 load balance 的角色.
许多开源的Edge Router(如 traefik, ambassador等)本身是支持 load balance 的, 在和 k8s 进行整合时配置的后端(上游)一般为 service.
在这种情况下 edge router(IngressController) 和 service 都`可以`发挥 load balance 作用. 这其中究竟是那个服务再起作用, 这是要分析的问题.

### #2. 提出假设:
* IngressController 没有发挥 lb 作用, 仅仅将请求透传给 Service 处理.
* IngressController 通过解析 service 域名获取 backend 短点.(service 为 headless 时 ingressController 可以行使 lb 功能; 其他情况 service 行使 lb 功能)
* IngressController 获取 Service 中的 selector 来 watch pods 变化, 并行使 lb 功能.
* IngressController 获取 Service 中的 endpoints 并监控其变化以行使 lb 功能.
* IngressController Watch 所有 pods 变化, 来行使 lb 功能

### #3. BASE
* 本文是基于 traefik:v2.0.5 来分析 lb 配置方式

### #4. 分析(原码按需大幅删减)
> 本人也没有 api-server/k8s-client 的实际编写经验, 下文会写如何从零去理解项目源码(欢迎交流源码阅读方式).

#### 首先实现 k8s 扩展有多种方式, 先确定项目实现方式. 打开 `go.mod`

```bash
# go.mod
k8s.io/api v0.0.0-20190718183219-b59d8169aab5
k8s.io/apimachinery v0.0.0-20190612205821-1799e75a0719
k8s.io/client-go v0.0.0-20190718183610-8e956561bbf5
k8s.io/code-generator v0.0.0-20190612205613-18da4a14b22b
```
基于此可以了解到项目通过`k8s.io/code-generator` ([github](https://github.com/kubernetes/code-generator))生成自定义资源的访问代码.

#### 查看 Makefile 来获取代码生成脚本:

```bash
# Makefile

generate-crd:
  ./script/update-generated-crd-code.sh

#./script/update-generated-crd-code.sh
"${REPO_ROOT}"/vendor/k8s.io/code-generator/generate-groups.sh \
  all \
  github.com/containous/traefik/${TRAEFIK_MODULE_VERSION}/pkg/provider/kubernetes/crd/generated \
  github.com/containous/traefik/${TRAEFIK_MODULE_VERSION}/pkg/provider/kubernetes/crd \
  traefik:v1alpha1 \
  --go-header-file "${HACK_DIR}"/boilerplate.go.tmpl \
  "$@"
```
可以发现 CRD 资源定义在 /pkg/provider/kubernetes/crd 下

#### 查看目录结构, 猜测 kubernetes/crd 是 provider 抽象的实现 
> 如果阅读过 traefik [文档]([provider](https://docs.traefik.io/providers/overview/))应该可以更快定位到这里,provider 提供了配置发现的功能.

```bash
$ tree -L 1 pkg/provider 
pkg/provider
├── docker
├── file
├── kubernetes
└── provider.go
```

#### 查看 provider.go 文件
```go
// provider.go
// Provider defines methods of a provider.
type Provider interface {
  // Provide allows the provider to provide configurations to traefik
  // using the given configuration channel.
  Provide(configurationChan chan<- dynamic.Message, pool *safe.Pool) error
  Init() error
}
```

文档注释中明确说明用来进行配置发现. 
以及定义了了两个方法 `Init` 和 `Provide`, 虽然没有注释,可以裸猜 init 方法进行了初始化. Provide 提供服务.

#### 在 /pkg/provider/kubernetes/crd 下查找 Provide 方法

```bash
$ ag 'func.*Provide' pkg/provider/kubernetes/crd
pkg/provider/kubernetes/crd/kubernetes.go
52:func (p *Provider) newK8sClient(ctx context.Context, labelSelector string) (*clientWrapper, error) {
85:func (p *Provider) Init() error {
91:func (p *Provider) Provide(configurationChan chan<- dynamic.Message, pool *safe.Pool) error {
168:func (p *Provider) loadConfigurationFromCRD(ctx context.Context, client Client) *dynamic.Configuration {
```
可以找到定义在 pkg/provider/kubernetes/crd/kubernetes.go 文件 91 行左右.

#### 分析 kubernetes.go 文件中的 Provide 方法.
> 对于 Provider (struct) 内字段, 我一般是不会提前看的, 如果没有备注的话很难猜出作用, 并且分分钟就会忘记, 仅当函数中用到了才去看一下, 然后选择性记录一下(在 go 语言里尤其要记录 chan)

先扫一眼 `Init` 方法, 以防在初始化中 hack 了内容.

```go
// pkg/provider/kubernetes/crd/kubernetes.go

// Init the provider.
func (p *Provider) Init() error {
  return nil
}
```
什么都没做, 开心

查看 `Provide` 方法(代码经过精简):

```go
// pkg/provider/kubernetes/crd/kubernetes.go

func (p *Provider) Provide(configurationChan chan<- dynamic.Message, pool *safe.Pool) error {
  k8sClient, err := p.newK8sClient(ctxLog, p.LabelSelector)

  pool.Go(func(stop chan bool) {
    eventsChan, err := k8sClient.WatchAll(p.Namespaces, stopWatch)

    for {
      select {
      case event := <-eventsChan:
        conf := p.loadConfigurationFromCRD(ctxLog, k8sClient)

        confHash, err := hashstructure.Hash(conf, nil)
        switch {
        default:
          p.lastConfiguration.Set(confHash)
          configurationChan <- dynamic.Message{
            ProviderName:  providerName,
            Configuration: conf,
          }
        }
      }
    }
  })
  return nil
}
```

由此可以整理(猜测)出大致的业务逻辑(之后会一步步验证)(函数名见名知意的重要性)

```bash
1. 通过newK8sClient生成 k8sclient
2. 进行资源监控 Watch, 并将对应的消息事件放入 eventsChan
3. 通过 loadConfigurationFromCRD 加载配置
4. 封装成配置信息dynamic.Message,并通过loadConfigurationFromCRD传递给外部
```

#### 先扫一眼`newK8sClient` 

最后生成了一个clientWrapper 实例, 很简单可以简单扫一眼

```go
// pkg/provider/kubernetes/crd/kubernetes.go

func (p *Provider) newK8sClient(ctx context.Context, labelSelector string) (*clientWrapper, error) {
  var client *clientWrapper
  client, err = newInClusterClient(p.Endpoint)

  return client, err
}

//--------------------------------------
// pkg/provider/kubernetes/crd/client.go
func newClientImpl(csKube *kubernetes.Clientset, csCrd *versioned.Clientset) *clientWrapper {
  return &clientWrapper{
    csCrd:         csCrd,
    csKube:        csKube,
    factoriesCrd:  make(map[string]externalversions.SharedInformerFactory),
    factoriesKube: make(map[string]informers.SharedInformerFactory),
  }
}

func createClientFromConfig(c *rest.Config) (*clientWrapper, error) {
  csCrd, err := versioned.NewForConfig(c)
  csKube, err := kubernetes.NewForConfig(c)

  return newClientImpl(csKube, csCrd), nil
}

func newInClusterClient(endpoint string) (*clientWrapper, error) {
  config, err := rest.InClusterConfig()

  return createClientFromConfig(config)
}
```

#### 查看 `WatchAll` 函数

```go
// pkg/provider/kubernetes/crd/client.go

func (c *clientWrapper) WatchAll(namespaces []string, stopCh <-chan struct{}) (<-chan interface{}, error) {
  // 定义一个 chan
  eventCh := make(chan interface{}, 1)
  // 将 chan 包装成一个 handle 方法
  eventHandler := c.newResourceEventHandler(eventCh)

  // 进行消息订阅
  for _, ns := range namespaces {
    factoryCrd := externalversions.NewSharedInformerFactoryWithOptions(c.csCrd, resyncPeriod, externalversions.WithNamespace(ns))
    factoryCrd.Traefik().V1alpha1().IngressRoutes().Informer().AddEventHandler(eventHandler)
    factoryCrd.Traefik().V1alpha1().Middlewares().Informer().AddEventHandler(eventHandler)
    factoryCrd.Traefik().V1alpha1().IngressRouteTCPs().Informer().AddEventHandler(eventHandler)
    factoryCrd.Traefik().V1alpha1().TLSOptions().Informer().AddEventHandler(eventHandler)
    factoryCrd.Traefik().V1alpha1().TraefikServices().Informer().AddEventHandler(eventHandler)

    factoryKube := informers.NewSharedInformerFactoryWithOptions(c.csKube, resyncPeriod, informers.WithNamespace(ns))
    factoryKube.Extensions().V1beta1().Ingresses().Informer().AddEventHandler(eventHandler)
    factoryKube.Core().V1().Services().Informer().AddEventHandler(eventHandler)
    factoryKube.Core().V1().Endpoints().Informer().AddEventHandler(eventHandler)

    c.factoriesCrd[ns] = factoryCrd
    c.factoriesKube[ns] = factoryKube
  }

  return eventCh, nil
}

func (c *clientWrapper) newResourceEventHandler(events chan<- interface{}) cache.ResourceEventHandler {
  // 包装成 resourceEventHandler 解决事件通知
  return &cache.FilteringResourceEventHandler{
    Handler: &resourceEventHandler{ev: events},
  }
}

type resourceEventHandler struct {
  ev chan<- interface{}
}
// 资源的创建, 更新和删除方法.
func (reh *resourceEventHandler) OnAdd(obj interface{}) { reh.ev <- obj }
func (reh *resourceEventHandler) OnUpdate(oldObj, newObj interface{}) { reh.ev <- obj }
func (reh *resourceEventHandler) OnDelete(obj interface{})  { reh.ev <- obj }
```

至此可以了解到 k8sClient 会了解到会订阅 crd 资源变化, 并将变化事件放入 eventsChan 等到处理.
结合上一步可以了解到 `loadConfigurationFromCRD` 是配置适配器, 也就是问题的关键点.

#### 查看`loadConfigurationFromCRD`(只看 http route 的发现过程, 其他资源类似)

```go
// pkg/provider/kubernetes/crd/kubernetes.go

func (p *Provider) loadConfigurationFromCRD(ctx context.Context, client Client) *dynamic.Configuration {
  conf := &dynamic.Configuration{
    HTTP: p.loadIngressRouteConfiguration(ctx, client, tlsConfigs),
  }

  cb := configBuilder{client}
  for _, service := range client.GetTraefikServices() {
    cb.buildTraefikService(ctx, service, conf.HTTP.Services)
  }

  return conf
}
```

#### 查看 loadIngressRouteConfiguration:

先提供一个 IngressRoute 的模板的例子.
```yaml
apiVersion: traefik.containo.us/v1alpha1
kind: IngressRoute
metadata:
  name: traefik-dashboard
  annotations:
    helm.sh/hook: "post-install"
  labels:
    app: traefik
    chart: traefik
spec:
  entryPoints:
    - web
  routes:
  - match: PathPrefix(`/dashboard`) || PathPrefix(`/api`)
    kind: Rule
    services:
    - name:  traefik-dashboard
      port: 80
  - match: PathPrefix(`/metrics`)
    kind: Rule
    services:
    - name:  traefik
      port: 8082
```

```go
// pkg/provider/kubernetes/crd/kubernetes_http.go

func (p *Provider) loadIngressRouteConfiguration(...) *dynamic.HTTPConfiguration {
  conf := &dynamic.HTTPConfiguration{}

  for _, ingressRoute := range client.GetIngressRoutes() {
    cb := configBuilder{client}
    for _, route := range ingressRoute.Spec.Routes {
      // 关心单个 service 关联的 pod 的 lb 配置方式 只看这一个分支即可
      if len(route.Services) == 1 {
        fullName, serversLB, err := cb.nameAndService(ctx, ingressRoute.Namespace, route.Services[0].LoadBalancerSpec)
        if serversLB != nil {
          // 这是重点, 接下来看 serversLB 的生成方式
          conf.Services[serviceName] = serversLB
        }
      }
    }
  }

  return conf
}
```

#### 查看 nameAndService

```go
// pkg/provider/kubernetes/crd/kubernetes_http.go

func (c configBuilder) nameAndService(...) (...) {
  serversLB, err := c.buildServersLB(namespace, service)

  return
}

/* 
svc LoadBalancerSpec 对应 yaml 文件中的 spec.routes.services 部分
如:
- name: traefik-dashboard
  port: 80
*/
type LoadBalancerSpec struct {
  Name      string          `json:"name"`
  Kind      string          `json:"kind"`
  Namespace string          `json:"namespace"`
  Sticky    *dynamic.Sticky `json:"sticky,omitempty"`

  Port               int32                       `json:"port"`
  Scheme             string                      `json:"scheme,omitempty"`
  HealthCheck        *HealthCheck                `json:"healthCheck,omitempty"`
  Strategy           string                      `json:"strategy,omitempty"`
  PassHostHeader     *bool                       `json:"passHostHeader,omitempty"`
  ResponseForwarding *dynamic.ResponseForwarding `json:"responseForwarding,omitempty"`

  Weight *int `json:"weight,omitempty"`
}

// buildServersLB creates the configuration for the load-balancer of servers defined by svc.
func (c configBuilder) buildServersLB(namespace string, svc v1alpha1.LoadBalancerSpec) (*dynamic.Service, error) {
  servers, err := c.loadServers(namespace, svc)

  lb := &dynamic.ServersLoadBalancer{}
  lb.Servers = servers

  return &dynamic.Service{LoadBalancer: lb}, nil
}

func (c configBuilder) loadServers(fallbackNamespace string, svc v1alpha1.LoadBalancerSpec) ([]dynamic.Server, error) {
  strategy := svc.Strategy

  if strategy != roundRobinStrategy {
    return nil, fmt.Errorf("load balancing strategy %s is not supported", strategy) // 挺尴尬的
  }

  namespace := namespaceOrFallback(svc, fallbackNamespace)

  service, exists, err := c.client.GetService(namespace, sanitizedName)

  confPort := svc.Port
  var portSpec *corev1.ServicePort
  // 取到了 service 中 port 部分定义
  for _, p := range service.Spec.Ports {
    if confPort == p.Port {
      portSpec = &p
      break
    }
  }

  var servers []dynamic.Server

  endpoints, endpointsExists, endpointsErr := c.client.GetEndpoints(namespace, sanitizedName)

  // 取到对应的 endpoints
  var port int32
  for _, subset := range endpoints.Subsets {
    for _, p := range subset.Ports {
      if portSpec.Name == p.Name {
        port = p.Port
        break
      }
    }

    protocol := httpProtocol
    scheme := svc.Scheme
    // 包装配置返回
    for _, addr := range subset.Addresses {
      servers = append(servers, dynamic.Server{
        URL: fmt.Sprintf("%s://%s:%d", protocol, addr.IP, port),
      })
    }
  }

  return servers, nil
}
```

### 结论

至此就了解到了 对于单个 svc traefik 是通过获取 endpoints 来行使 lb 功能.(仅支持RoundRobin算法)

### 后续
如果有兴趣可以去看下traefik 的启动过程. 来看看是不是就如推测那样调用了 Provider.Provide 方法. 这一篇到这里结束.