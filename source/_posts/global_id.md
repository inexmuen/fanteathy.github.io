---
title: 分布式系统中的全局唯一ID
date: 2017-01-20 17:00:00
author: Fanteathy
tags: arch
categories: arch
---

在分布式系统中，生成全局唯一ID的需求很常见。生成的方法主要有`基于数据库生成`，`使用分布式集群协调器`和`划分命名空间并行生成`等。

## 常用技术方案

### 基于数据库生成

这种类型的生成方案，都可以设置其`初始值`以及`增量步长`，一般包含以下几种:

- 使用mysql auto_increment特性
- 使用postgresql sequence特性
- 使用oracle的sequence特性

为了避免数据库的单点故障，还可以使用多台数据库服务器使用不同的步长来生成ID

各种基于数据库的全局唯一ID生成方案可以参考: [http://www.cnblogs.com/heyuquan/p/global-guid-identity-maxId.html](http://www.cnblogs.com/heyuquan/p/global-guid-identity-maxId.html)

### 基于分布式集群协调器生成

在不使用数据库的情况下，通过一个后台服务对外提供高可用的、固定步长标识生成，则需要分布式的集群协调器进行。

一般的，主流协调器有两类：

- 以强一致性为目标的：ZooKeeper为代表

- 以最终一致性为目标的：Consul为代表

ZooKeeper的强一致性，是由Paxos协议保证的；Consul的最终一致性，是由Gossip协议保证的。

在步长累计型生成算法中，最核心的就是保持一个累计值在整个集群中的「强一致性」。但是，这也会为唯一性标识的生成带来新的形成瓶颈。

### 划分命名空间并行生成

似乎对于分布式的ID生成，以Twitter Snowflake为代表的， Flake 系列算法，经常可以被搜索引擎找到，但似乎MongoDB的ObjectId算法，更早地采用了这种思路。

snowflake是Twitter开源的分布式ID生成算法，结果是一个long型的ID。其核心思想是：使用41bit作为毫秒数，10bit作为机器的ID（5个bit是数据中心，5个bit的机器ID），12bit作为毫秒内的流水号（意味着每个节点在每毫秒可以产生 4096 个 ID），最后还有一个符号位，永远是0。snowflake算法可以根据自身项目的需要进行一定的修改。比如估算未来的数据中心个数，每个数据中心的机器数以及统一毫秒可以能的并发数来调整在算法中所需要的bit数。

## 我的Global Id

用于标识电商网站订单号

### 特性

- 12-19位int类型。方便用户阅读使用，同时数据库长整数字型的数据索引与检索效率，远远高于文本型。(Java long型对应 2 ^ 64为20位10进制型，所以也在long范围内)
- 需要标识业务/技术属性
	- 业务属性 
		- 订单类型，如实物商品/虚拟商品
		- 外卖订单/团购订单
		- ...
	- 技术属性 
		- IDC归属
		- 测试订单
		- ...
- 考虑到长度问题，不考虑包含时间因素
- 包含卖家/买家信息

### 设计

```
[自增部分] + [1位业务标识] + [1位技术标识] + [3位商家标识] + [3位用户标识]
```

- 自增部分用来保证订单号的唯一以及订单号的有序，使用DB自增生成
- 业务标识用来标识业务属性
- 技术标识用来标识技术属性
- 商户ID散列值 mod 512，可以用来做商户维度的sharding，根据笔者经验，大中型网站分片数512已经足够。同时不考虑2/10进制转换来标识sharding，也是为了阅读方便
- 用户ID散列值 mod 512，可以用来做用户维度的sharding

### 容量评估

假设订单长度大于12位(后8位为标识位)，则自增的起始值为1,000。到20位时，自增值为1000,0000,0000。以1000,0000单一天计算，则订单号可以使用

```
(100000000000 - 1000) / 10000000 / 365 = 27(年)
```

### 存储及使用

自增的起始值存储在mysql中，使用的过程如下:

```
queue = multiprocessing.Queue()

def get_global_id():
    order_id = queue.get()
    if order_id:
        // fetch from queue
        return order_id
    else:
        // fetch from db
        order_id_list = fetch_id_list_from_db()
        for order_id in order_id_list:
            queue.put(order_id)
        order_id = queue.get()
        return order_id
        
def fetch_id_list_from_db():
    db.select_for_update() // db获取悲观锁，注意此时表只有一行，所以需要通过ID获取行锁，忌使用表锁
    order_ids = db.select(1000) // 一次取1000个order id
    db.sequence.update(1000) // db sequence 字段增加1000
    db.commit()
    return order_ids
```