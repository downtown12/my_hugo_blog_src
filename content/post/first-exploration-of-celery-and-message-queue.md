---
title: "初探：Python Celery任务调度模块和消息队列"
date: 2018-05-10T20:14:36+08:00
draft: false
description: "Example article description"
# thumbnail: "img/placeholder.jpg" # Optional, thumbnail
disable_comments: false # Optional, disable Disqus comments if true
categories:
  - "开发"
tags:
  - "Python"
  - "任务调度"
  - "消息对列"
  - "MQ"
---

近期由于工作需要，要开发一套类似扫描器的工具，并将它的扫描工作作为服务提供给多个用户使用。每个用户可以下发扫描任务，最终查看扫描结果。为了实现调度多个用户下发的扫描任务。我想到需要写一个同步或异步的任务调度模块，并要将用户下发的任务放进队列里，依次（同步）或并发（异步）处理。同时，队列里的任务要存放在缓存（或本地磁盘）中，以免任务信息丢失。
>为了实现这几个功能，上网查了些资料，发现消息队列技术刚好可以完成这项工作。因此也就不需要自己研发这个模块了。刚好前不久也被[lijiejie](http://www.lijiejie.com/)大牛问到了类似的问题。之前没有接触过。正好学了一下，发现使用现成的任务调度模块（Celery）以及消息队列（RabbitMQ或Redis）来实现文章一开始所说的任务调度功能，其实很简单。

## 参考连接
1. [任务调度利器：Celery (廖雪峰)](https://www.liaoxuefeng.com/article/00137760323922531a8582c08814fb09e9930cede45e3cc000)
2. [利用 Celery 构建 Web 服务的后台任务调度模块(ibm-developworks)](https://www.ibm.com/developerworks/cn/opensource/os-cn-celery-web-service/index.html)

## Celery是什么

> Celery是Python开发的分布式任务调度模块，接口简单，开发容易
> Celery本身不含消息服务，它使用第三方消息服务来传递任务，目前，Celery支持的消息服务有RabbitMQ、Redis甚至是数据库。

对与我目前需求比较简单的情况，当然Redis应该是最佳选择。后续随着需求复杂，可以考虑使用RabbitMQ（分布式集群的消息队列）

前边几段提到了RabbitMQ和Redis, 这两个东西在任务调度里的作用是什么呢？
答案是，他们提供了消息队列服务。
简单来讲，它提供了任务存放的位置和存取方式。这样，在任务处理模块有空闲的时候，就从队列里取出一个任务来执行，执行完了（或中场休息）再取出另外一个。
消息队列在任务下发者（生产者）和任务处理者（消费者）之间提供了沟通的桥梁。

## 消息队列的三个重要组件
* Producer（客户端）
* Broker（中间人）
* Worker（职程）

消息队列的输入是工作的一个单元，称为任务，独立的**职程（Worker）**进程持续监视队列中是否有需要处理的新任务。
Celery 用消息通信，通常使用**中间人（Broker）**在**客户端**和**职程**间斡旋。这个过程从客户端向队列添加消息开始，之后中间人把消息派送给职程。

## Broker的选择：Redis还是RabbitMQ
简单描述如下：

**Redis**属轻量级消息队列，支持功能不多，不支持持久化任务。

**RabbitMQ**属重量级消息队列，支持持久化任务。
