---
layout: post
comments: true
title: 'Airflow On K8S'
date: 2019-01-31
author: Calvin Wang
tags: airflow k8s
---

### 问题
原先公司的Airflow是基于LocalExecutor的一个单机应用,  随着业务发展任务数不断增加, 导致单机性能不断进行升级. 但是数仓任务的特殊性(集中于凌晨开始运行), 白天机器有大量资源被浪费. 基于此将Airflow以容器化的方式进行部署. 来实行Scheduler可用和Woker组的自动扩缩容.


### 容器化技术
简单地说就是将 `程序` 和 `程序所需的依赖` 组装好 放到`盒子`了.  以后需要的时候将`盒子`拿出来用就好了, 不需要重新组装了.
他呢由如下几个特点:

* 统一,简单: 容器化结果就是一个个的`小盒子`, 可以讲这些`盒子`用在各个环境(产品,测评,开发), 保证的环境的一致性.
  举个例子:
    团队每来一个小伙伴都需要配置其开发环境, 由于文档化工作不好, 每个小伙伴的开发环境和线上环境都有点点不同(比如: 开发环境Airflow版本是1.7.1.3 但是小伙伴安装的确是1.8 / 1.10等), 然后产生一些不必要的兼容问题.
  利用docker的解决方案:
  ```bash
  # 登陆公司仓库
  docker login registry.git.saybot.net 
  
  # 下载Airflow镜像(小盒子)
  docker pull registry.git.saybot.net/data-warehouse/docker-airflow/docker-airflow:1.10.1
  
  # 先启动来看看效果
  docker run -P registry.git.saybot.net/data-warehouse/docker-airflow/docker-airflow:1.10.1
  
  # 把咱们自己的程序(DAG) 应用到镜像上
  ### 先生成一个demo DAG file
  cat << EOF > /tmp/demo.py
  import airflow
  from airflow import DAG
  from datetime import timedelta
  from airflow.operators.bash_operator import BashOperator
  
  default_args = {
      'owner': 'airflow',
      'depends_on_past': False,
      'start_date': airflow.utils.dates.days_ago(2),
      'email': ['airflow@example.com'],
      'email_on_failure': False,
      'email_on_retry': False,
      'retries': 1,
      'retry_delay': timedelta(minutes=5),
  }
  
  dag = DAG(
      'demo',
      default_args=default_args,
      description='A simple tutorial DAG',
      schedule_interval=timedelta(days=1),
  )
  
  t1 = BashOperator(
      task_id='print_date',
      bash_command='date',
      dag=dag,
  )
  EOF
  ### 将新产生的DAG运行起来
  docker run -d -v /tmp/demo.py:/usr/local/airflow/dags/demo.py -P registry.git.saybot.net/data-warehouse/docker-airflow/docker-airflow:1.10.1

  ```
* 轻量(快速部署): 一个组装好的小盒子 相比于 需要从头组装小盒子(比如EMR)启动起来那一定要快的. 因为容器化技术让小盒子公用了系统资源, 相比虚拟化(ec2)不需要虚拟化系统, 那启动速度也要快的. 启动快也就导致了部署快.
* 充分利用资源
* 应用隔离


### Airflow 镜像构建:
公司使用的Airflow镜像由两层组成, 第一层是一个base airflow: 用来解决Airlow依赖的问题, 构建一个最小的可用的airflow镜像. 第二层 是基于第一层镜像进行修改, 做一些公司定制, 如修改认证, 配置邮箱服务等.
![IMAGE](/assets/img/20190131/83DB99F0F2E862CC4C86574CAD9941CE.jpg)

### Airflow 部署流程:
![IMAGE](/assets/img/20190131/D66DBFA485AAF6E60E10E1BAC87AA4FB.jpg)

### DW 项目配置:

* 项目DAG files `装箱`
{% highlight dockerfile %}
# Dockerfile

FROM alpine:3.4

COPY airflow /usr/local/airflow-projects/dw-code
{% endhighlight %}

* 去airflow deploy项目中配置DW项目

{% highlight yaml %}
# _initContainers.yaml

- name: dw-code
  image: "registry.git.saybot.net/data-warehouse/dw/\{\{ .Values.project.env \}\}:latest"
  imagePullPolicy: \{\{ .Values.image.pullPolicy \}\}
  command:
    - /bin/sh
    - '-exc'
  args:
    - 'mv -f /usr/local/airflow-projects/dw-code \{\{ .Values.dags.dag_path \}\}/;
      mkdir -p \{\{ .Values.dags.dag_path \}\}/dw-code/config ;
      for conf in /tmp/dw-code/config/*; do cat $conf >> \{\{ .Values.dags.dag_path \}\}/dw-code/config/`basename $conf`; done;
      '
  volumeMounts:
    - name: dags-data
      mountPath: \{\{ .Values.dags.dag_path \}\}
    - name: airflow-secret
      mountPath: /tmp/dw-code/config
{% endhighlight %}

* 配置CI进行持续集成

```yaml
# .gitlab-ci.yml

stages:
  - build # build image
  - deploy # trigger airflow deployment

variables: &VARIABLES
  IMAGE_PER_BRANCH_COMMIT: $CI_REGISTRY_IMAGE/$CI_COMMIT_REF_NAME:$CI_BUILD_REF

.default: &BUILD
  image: docker:latest
  stage: build
  services:
    - name: docker:dind
      command: ["--registry-mirror", "https://ixceb9no.mirror.aliyuncs.com"]
  variables: &VARIABLES
    DOCKER_DRIVER: overlay2
    IMAGE_PER_BRANCH: $CI_REGISTRY_IMAGE/$CI_BUILD_REF_NAME:latest
  before_script:
    - docker login -u "$CI_REGISTRY_USER" -p "$CI_REGISTRY_PASSWORD" $CI_REGISTRY
  script:
    - echo $IMAGE_PER_BRANCH_COMMIT
    - docker pull ${IMAGE_PER_BRANCH} || true
    - docker build --pull --cache-from ${IMAGE_PER_BRANCH} -t ${IMAGE_PER_BRANCH_COMMIT} -t ${IMAGE_PER_BRANCH} --build-arg CI_JOB_TOKEN=$CI_JOB_TOKEN .
    - docker push ${IMAGE_PER_BRANCH_COMMIT}
    - docker push ${IMAGE_PER_BRANCH}
  tags:
    - docker
  except:
    - tags

build_dev:
  <<: *BUILD
  only:
    - dev

.deploy: &deploy
  stage: deploy
  image: appropriate/curl:latest
  script:
  - curl --request POST --form "token=$CI_JOB_TOKEN" --form "ref=$AIRFLOW_CI_BRANCH" https://git.saybot.net/api/v4/projects/1711/trigger/pipeline
  tags:
    - docker

deploy_dev:
  <<: *deploy
  variables:
    AIRFLOW_CI_BRANCH: k8s-ci
  only:
    - dev
```
