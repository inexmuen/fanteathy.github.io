---
title: 延时任务
date: 2016-10-10 22:00:00
author: Fanteathy
tags: arch
categories: arch
---

## 背景

应用中都有延时任务的需求，需要任务在指定的时间之后执行，这里讨论一下延时任务的技术实现。

## 实现方式

实现方式主要有以下几种:

- `rabbitmq + dead letter exchange`或者`rabbitmq + exchange plugin: x-delayed-message`。利用queue的`x-message-ttl`控制消息延时时间
- DB轮询。将task存储在DB中，并设置task的执行时间，同时轮询DB取出当前需要执行的task执行。优点是可结合DB事务，缺点是可能存在单点问题
- rabbitmq存储task，consumer延时处理。consumer端sleep N秒。当N较大时，mq中可能会堆积大量消息
- 使用beanstalkd, nsq等天生很好支持延时特性的queue，但是使用较少，社区缺少相应的技术支持

## dead letter exchange

此处介绍一下DLX的创建, DLX的整体流程如下:

```
delay_exchange => delay_queue => biz_exchange => biz_queue
```

创建流程如下:

- 创建延时exchange: `delay_exchange`
- 创建延时queue: `delay_queue`
	- 设置延迟时间: `x-message-ttl`
	- 设置消息到达expire时间后转到的exchange: `x-dead-letter-exchange`，需要为真正的业务exchange(`biz_exchange`)
- 设置`delay_queue`和`delay_exchange`的绑定
- 创建业务exchange: `biz_exchange`
- 创建业务queue: `biz_queue`
- 设置`biz_exchange`和`biz_queue`的绑定

此时发送到`delay_exchange`的消息，会在`x-message-ttl`设置的时间后，经过`delay_queue`转到`biz_exchange`，即到`biz_queue`里面。


