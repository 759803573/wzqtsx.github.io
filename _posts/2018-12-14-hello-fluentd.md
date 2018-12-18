---
layout: post
comments: true
title: 'Hello Fluentd'
date: 2018-12-14
author: Calvin Wang
cover: "/assets/img/fluentd/fluentd.jpg"
tags: fluentd
---

> 由于工作原因接触到了fluentd, 尝试写写分享, 锻炼一下文档能力. 计划写两篇分享: 第一篇介绍fluentd, 第二篇分析fluentd源码. 本文是第一篇: Hello FLuentd!


## 简介
fluentd 被设计为通过插件系统来连接各个系统, 也就是利用插件构建一个`管道`将数据安全可靠的从 Source 拷贝至 Destination.

### 典型是使用方式:
  * 将数据采集至可靠,廉价的外部系统(如: s3,hdfs)
  * 过滤特定数据并发送至警报系统
  * Stream Processing

##  插件系统
一个典型的`管道`如下图所示:

![Fluentd-Pipeline](/assets/img/fluentd/pipeline.png)

### `InputPlugin`: 通过 Poll/Push 模式获取数据
  * 通过同一个 `InputPlugin`采集到的数据会增加相同的`tag`. 
  * `tag`: 用来标记数据来自哪里,并得到数据该流去哪里.

### `ParserPlugin`: 将数据解析成 `EStream`
  * 每个 ESStream 是由`time`, `record` 两部分组成

eg.

```ruby
# CSV ParserPluginConfig
<parse>
  @type csv
  keys time,host,req_id,user_id,user_type # csv header 信息
  time_key time # 指明 header 中的 time 字段
</parse>

# Input Data
2013/02/28 12:00:00,192.168.0.1,111,123,passport_id

# Output
time:
1362020400 (2013/02/28/ 12:00:00)

record:
{
  "host"   : "192.168.0.1",
  "req_id" : "111",
  "user_id": "123",
  "user_type": "passport_id"
}
```

至此每个Event就有3部分(`field`)组成: `tag`, `time`, `record`.

### `FilterPlugin`: 对`EStream`数据进行过滤,修改
  * 通过对`flied(s)`判断来过滤数据(如: 仅通过特定级别日志)
  * 通过增加新的字段来丰富数据(如: 通过 ip 增加地理位置信息)
  * 删除或者屏蔽部分字段来满足日志规范(如: 屏蔽密码)

### `FormatPlugin`: 定制输出格式
* 默认情况下 Plugin 会以```time<TAB>tag<TAB>record```作为输出格式. 

eg. 

```text
2014-08-25 00:00:00 +0000 foo.bar<TAB>{"k1":"v1", "k2":"v2"}) 
```

一般情况下默认格式都不是最终想要的数据格式, 这个时候可以通过`FormatPlugin`插件来对输出格式进行重新定义.

eg.

```ruby
# CSV FormatPlugin Config:
<format>
  @type csv
  fields host,method
</format>

# Event:
tag:    app.event
time:   1362020400
record: {"host":"192.168.0.1","size":777,"method":"PUT"}

# Output:
"192.168.0.1","PUT"
```

### `BufferPlugin`: 发送前暂存数据
  * Chunk: 是一组数据(format 插件output的数据)的集合
  * 具有两个存储位置
    * stage: 未装满的 Chunk 所处的位置
    * queue: 对于已装满的 Chunk 会移动至这里等待被 OutputPlugin 送出
  * chunk keys: 用来控制 Chunks 的生成

eg.

```ruby
# BufferPlugin Config
<buffer time>
  timekey      1h # 一个 Chunk 最多保留一小束数据
  timekey_wait 5m # 延迟5min 发送(用来处理延迟事件)
</buffer>


# 生成的 Chunks

11:59:30 web.access {"key1":"yay","key2":100}  ------> CHUNK_A

12:00:01 web.access {"key1":"foo","key2":200}  --|
                                                 |---> CHUNK_B
12:00:25 ssh.login  {"key1":"yay","key2":100}  --|
```

### `OutputPlugin`: 将数据发送至目的系统
  * 占位符: 用 chunk key 替换占位符
  * 三种发送模式:
    * None-Buffered(#process): 不使用缓存直接发送至目的系统,一般不处理错误(至多一次)
    * Synchronous-Buffered(#write): 利用缓存,已同步的方式发送至目的系统(恰好一次)
  ![IMAGE](/assets/img/fluentd/synchronous_buffered.jpg)
    * Asynchronous-Buffered(#try_write): 利用缓存, 以异步的方式发送至目的系统(至少一次)
  ![IMAGE](/assets/img/fluentd/asynchronous_Buffered.jpg)
  * 模式选择:(对于实现的方法,依据缓存偏好设置来决定发送模式)
  ![IMAGE](/assets/img/fluentd/work-flow.jpg)

## k8s 日志转储至 s3

```ruby
# Config
<source>
  @type tail
  @id in_tail_container_logs
  path /var/log/containers/*.log
  pos_file /var/log/fluentd-containers.log.pos
  tag kubernetes.*
  read_from_head true
  <parse>
    @type json
    time_format %Y-%m-%dT%H:%M:%S.%NZ
  </parse>
</source>

<filter kubernetes.**>
  @type kubernetes_metadata
  @id filter_kube_metadata
</filter>

<match>
  @type s3
  @id k8s_out_s3
  @log_level info
  s3_bucket "#{ENV['FLUENT_S3_BUCKET_NAME']}"
  s3_region "#{ENV['FLUENT_S3_BUCKET_REGION']}"
  aws_key_id "#{ENV['FLUENT_S3_AWS_KEY_ID']}"
  aws_sec_key "#{ENV['FLUENT_S3_AWS_SEC_KEY']}"
  path k8s/%Y%m%d/${$.kubernetes.namespace_name}/${$.kubernetes.pod_name}/${$.kubernetes.container_name}/${tag[4]}
  s3_object_key_format %{path}-%{index}.%{file_extension}
  <buffer tag,time,$.kubernetes.namespace_name,$.kubernetes.pod_name,$.kubernetes.container_name>
    @type file
    path /var/log/fluentd-buffers/s3.buffer
    timekey 1h
    timekey_wait 10m
    flush_at_shutdown true
  </buffer>
  <format>
    @type single_value
    message_key log
  </format>
</match>

### s3日志
~ ⌚ 1:27:56
$ aws s3 ls s3://logs-k8s/dev-k8s/20181213/kube-system/calico-node-fm594/calico-node/
2018-12-13 16:15:05      33514 calico-node-fm594_kube-system_calico-node-8c70f3c49313bb6095b2fd64e260a0157e04521ac006386bd8adfb3fc8689a62-0.gz
2018-12-13 17:15:04      18031 calico-node-fm594_kube-system_calico-node-8c70f3c49313bb6095b2fd64e260a0157e04521ac006386bd8adfb3fc8689a62-1.gz
2018-12-13 18:15:03      18003 calico-node-fm594_kube-system_calico-node-8c70f3c49313bb6095b2fd64e260a0157e04521ac006386bd8adfb3fc8689a62-2.gz
2018-12-13 19:15:03      17919 calico-node-fm594_kube-system_calico-node-8c70f3c49313bb6095b2fd64e260a0157e04521ac006386bd8adfb3fc8689a62-3.gz
2018-12-13 20:15:03      17977 calico-node-fm594_kube-system_calico-node-8c70f3c49313bb6095b2fd64e260a0157e04521ac006386bd8adfb3fc8689a62-4.gz
2018-12-13 21:15:04      17870 calico-node-fm594_kube-system_calico-node-8c70f3c49313bb6095b2fd64e260a0157e04521ac006386bd8adfb3fc8689a62-5.gz
2018-12-13 22:15:03      17929 calico-node-fm594_kube-system_calico-node-8c70f3c49313bb6095b2fd64e260a0157e04521ac006386bd8adfb3fc8689a62-6.gz
2018-12-13 23:15:03      18012 calico-node-fm594_kube-system_calico-node-8c70f3c49313bb6095b2fd64e260a0157e04521ac006386bd8adfb3fc8689a62-7.gz
```

<hr />

待续

<hr />

### 附件
* 脑图![Fluentd Plugin.png](/assets/img/fluentd/brain_map.png)