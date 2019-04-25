---
layout: post
title: 'Hello Redis'
date: 2019-04-23
author: Calvin Wang
#cover: 'http://on2171g4d.bkt.clouddn.com/jekyll-banner.png'
tags: Redis
---

> Alo7数仓内部分享, 以下如有任何不妥之处, 恳请告知.


## Redis 介绍
Redis 是一个开源（BSD许可）的，内存中的数据结构存储系统，它可以用作数据库、缓存和消息中间件. 它支持多种类型的数据结构，如 字符串（strings）， 散列（hashes）， 列表（lists）， 集合（sets）等.(翻译自[官网](https://redis.io/topics/introduction))

下面先讲最常用的缓存的功能.

## 缓存的出现(以 BI 日志为例)(BI只是用来举例不代表他是正确的做法)

### 背景:
BI日志系统将行为数据分为两部分来进行采集.
分别为:
  * 基础信息: 在应用使用过程中不变的信息, 如设备, 用户, 应用自有信息等.
  * 事件信息: 一个事件特有信息, 如事件名, 事件参数等.
BI Server 会将基础信息保存至DB, 之后收到事件信息再去查DB获取到基础信息, 然后拼装成一个完整事件, 推送到Queue.

### 存在的问题:
如下图: 一般一个请求可能是由零到一个基础信息配置和多个事件信息组合而成的. Server处理Request会遍历每一个事件, 基础信息保存DB, 事件信息去DB查询并补全, 然后塞队列. 这其中因为同一个Resquest的多个事件一般具有相同的基础信息, 反复查DB, 耗时增加, 而且徒增DB压力.

![question1](/assets/img/201904/hello-redis-1.png)

### 优化方案:
如下图: 最简单的优化, 在Server中开辟一块内存空间来保存基础信息, 处理事件信息时, 先去内存中查找, 找不在再去DB中查找, 并更新在内存中.

![R1](/assets/img/201904/hello-redis-1-1.png)

### 对, 又出现问题了(例子和实际有一些改动, 方便解释)
如下图: 为了高可用Server一般会启动多台, 假设请求顺序是Request1, Request2, Request3. 业务长时间客户端的基础信息在Request完成后发生了改变, 在Request2中进行更新. Request3被路由到了处理Request1的Server 导致取到了Request1的基础信息,导致了不一致.

![question2](/assets/img/201904/hello-redis-2.png)

### 再一次优化方案:
如下图: 问题是由于每个Server自己开辟了内存空间来Cache数据, 导致了不一致发生, 那就把Cache扔到Server之外就好了.

![R2](/assets/img/201904/hello-redis-2-1.png)

## 几个问题出现了

### 问题1: 在上诉场景中, 引入Cache的目的是为了加快处理速度,并减轻了DB压力. 但是引入了外部Cache以后为什么会加快处理速度呢?

##### 回答1: Cache 把数据存在内存里, 所以快

##### 追问1: 像Mysql使用Innodb 引擎, 当数据只有1G, 而内存打到了4G(反正就是数据比内存小), 这时的Mysql就可以充当为一个内存 DB, 速度为什么比 Cache 慢?

##### 回答2: 
* 逻辑架构

MySQL逻辑架构如下图,[引用自](https://www.rathishkumar.in/2016/04/understanding-mysql-architecture.html): 一个SQL的执行, 需要 建立连接, 认证, 鉴权, [检查缓存], 解析, 优化, 调用存储引擎(这个里面也有很多东西)

![MySQL Architecture](/assets/img/201904/mysql-architecture.png)

Redis逻辑架构: 建立连接, 认证, 执行.

* 特长不同, [参考自](https://www.quora.com/Is-NoSQL-faster-than-SQL)

Redis这类NoSQL db设计去解决一些简单的查询, 如get,set, sort等.
但是灵活性相比SQL稍有不足, 需要去设计数据存储结构. 

eg. 网站有文章, 文章有评论. 要保存这样的数据在SQL DB/MySQL 中, 可能会建立两个实体: posts, comments. 在No SQL/Redis 中 就可以把 post 和 comments 保存成一个文档`{"post":"...","comments":[...]}`, 以post唯一ID作为key来保存. 使用时, SQL DB/MySQL 需要关联表查结构, NoSQL/Redis 直接Get 就好了. 不过在此之上如果想在Server 端实现类型朋友圈的功能(只看好友/关注)的评论. 应该SQL DB/MySQL实现起来会更快一些(灵活性)

### 缓存雪崩(接下来这三个是我强行举例了, 我没有真实遇上过)
4:30 大量用户登录线上教室开始上课, Server 基本基础信息, 并设置缓存为1h的有效期(这里涉及到了一个淘汰机制的概念,后面讲. 这里可以理解了,有效期到了内存就不可用了)

随着业务高峰的来临越来越多的用户加入系统.

5:30 当天第二节课开始, 大量用户登录线上教室开始上课, 于此同时第一节课的用户下课了, 于此同时第一节课用户的缓存全过期失效了. 

大量的请求发送到了DB去更新基础信息.

嗯, DB挂了...

以上: 同时大量缓存失效, 致使DB压力剧增, 叫做缓存雪崩.

解决方案: 有效期加一个随机salt

### 缓存穿透

在上述例子中,  会议session_id 作为key 来保存基础信息. 

流程:

1. 拿session_id 去 cache 里查找, 找到直接使用
2. 找不到 去db取基础信息
3. 取到了 更新 缓存, `取不到跳过`.

问题出在了3这一步上, `取不到跳过`, client端发生了bug/也可能是恶意攻击, 导致 基础信息和事件信息的session_id 匹配不上, 导致了大量请求全部跳过Cache 全部查DB, 而查询结果继续为NULL, Cache 层起不到应有的作用(就和没有一样), 嗯 db 又挂了...

以上: 利用大量无效KEY进行查询, 导致DB压力剧增. 叫做: 缓存穿透

解决方案: 查不到也保存在Cache中

### 缓存击穿

在上述例子中, client 又有bug了.

1. client 正常提交了基础信息
2. 基础信息Cache失效
3. (BUG开始) 事件信息以1条为单位, 向Server发送
4. Server 同时收到大量具有相同session_id的事件信息
5. Server 以session_id同时向Cache查询, 同时查不到
6. Server 同时去DB查询
7. 嗯, db又挂了.

以上: 大量请求访问统一失效的key, 致使DB压力升高的行为, 叫: 缓存击穿

解决方案: 利用锁机制

## Redis 数据类型(强烈建议去看[官方文档](https://redis.io/commands))(示例都是在[Try Redis](http://try.redis.io/)上进行的):
![redis-data](/assets/img/201904/redis-data.png)

* String
![redis-string](/assets/img/201904/redis-string.png)

* Hash
![redis-hash](/assets/img/201904/redis-hash.png)

* List: 队列的功能
![redis-list](/assets/img/201904/redis-list.png)

* Set
![redis-set](/assets/img/201904/redis-set.png)

* Sorted Set
![redis-sorted_set](/assets/img/201904/redis-sorted_set.png)

CAT定时报告就是利用Soted Set 实现的. 将需要发送报告的的内容作为value, 发送时间戳作为score 存入redis. Sender定期扫描Redis相关时间范围的记录, 并发送

待续:

* 发布/订阅
* 淘汰策略
* 集群模式
* 数据持久化
* Pipeline
