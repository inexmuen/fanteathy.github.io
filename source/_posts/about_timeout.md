---
title: 关于timeout
date: 2017-01-16 15:00:00
tags: arch
categories: arch
---

## 使用

`Timeout`，是一种常见的容错模式。常见的有设置网络连接超时时间，一次RPC的响应超时时间等。在分布式服务调用的场景中，它主要解决了当依赖服务出现建立网络连接或响应延迟，不用无限等待的问题，调用方可以根据事先设计的超时时间中断调用，及时释放关键资源，如Web容器的连接数，数据库连接数等，避免整个系统资源耗尽出现拒绝对外提供服务这种情况。

## 原理

### 阻塞IO的timeout

由于我们常用的socket都是阻塞的，那么超时是很好设置的，直接设置socket的超时就可以了。

```
// send timeout
setsockopt(socket，SOL_S0CKET,SO_SNDTIMEO，(char *)&nNetTimeout,sizeof(int));
// receive timeout
setsockopt(socket，SOL_S0CKET,SO_RCVTIMEO，(char *)&nNetTimeout,sizeof(int));
```

### 非阻塞IO的timeout

但是，用过Linux下非阻塞I/O的都知道，非阻塞情况下，设置连接超时神马都是浮云的，因为人家是非阻塞的。典型的譬如Python的gevent，在用monkey.patch_all()之后，所有的socket都会被转化为非阻塞的，这时候的timeout设置就失效了。这种情况下gevent提供了Timeout类，当你的类似sleep（由于I/O、sleep等原因挂起）超时超过了Timeout的时间限制后，会自动终止block，跳出。使用方法如下:

```
import gevent
import gevent.monkey
import time

gevent.monkey.patch_all()

def test():
    with gevent.Timeout(5) as timeout:
        time.sleep(10)
        print "time out"

if __name__ == "__main__":
    g = gevent.spawn(test)
    g.join()
```

## 最佳实践

为了让client尽快得到响应，也为了尽量减少服务响应延迟时的服务资源消耗，需要设置`client timeout(客户端timeout)`, `server timeout(服务端timeout)`以及`service timeout(基础服务如mysql等timeout)`。同时需要根据实际情况设置一条请求链路上对应client,server和service的timeout。

`ToDo`

## mysql timeout实验

目的: 实验client通过RPC请求SERVER，同时操作MYSQL时相应timeout的最佳设置

### 背景

table结构如下:

```
<resultMap id="BaseResultMap" type="me.ele.fin.job.dal.model.TestModel">
        <id column="id" property="id" jdbcType="BIGINT"/>
        <result column="restaurant_id" property="restaurantId" jdbcType="BIGINT"/>
        <result column="order_id" property="orderId" jdbcType="BIGINT"/>
        <result column="status" property="status" jdbcType="TINYINT"/>
        <result column="created_at" property="createdAt" jdbcType="TIMESTAMP"/>
        <result column="updated_at" property="updatedAt" jdbcType="TIMESTAMP"/>
    </resultMap>
```

对应RPC请求接口逻辑如下:

```
    @Transactional(timeout = 5)
    public int updateTestStatus(Long id) throws JobServiceException {
        Long startTime = System.currentTimeMillis();
        int updateRes = 0;
        try {
            updateRes = testMapper.updateById(id);
        } catch (Exception e) {
            e.printStackTrace();
        }
        Long endTime = System.currentTimeMillis();
        String duration = DurationFormatUtils.formatPeriod(startTime,
                endTime, "S");
        logger.info("update result: " + updateRes);
        logger.info("Duration milliseconds: " + duration);
        return 1;
    }
```

mysql对应timeout设置如下:

```
mysql> show variables like "%timeout%";
+-----------------------------+----------+
| Variable_name               | Value    |
+-----------------------------+----------+
| connect_timeout             | 10       |
| delayed_insert_timeout      | 300      |
| innodb_flush_log_at_timeout | 1        |
| innodb_lock_wait_timeout    | 50       |
| innodb_rollback_on_timeout  | OFF      |
| interactive_timeout         | 28800    |
| lock_wait_timeout           | 31536000 |
| net_read_timeout            | 30       |
| net_write_timeout           | 60       |
| rpl_stop_slave_timeout      | 31536000 |
| slave_net_timeout           | 3600     |
| wait_timeout                | 28800    |
+-----------------------------+----------+
12 rows in set (0.00 sec)
```

### 步骤

#### 1. 设置`autocommit`为OFF

```
set autocommit = 0;
```

#### 2. 锁定某一条记录

```
begin;
select * from test where id = 1 for update;
```

锁定id为1的记录

#### 3. 调用接口，update同一条记录

运行以下JUnit测试

```
public class JobServiceTest extends TestBase {

    @Autowired
    private IJobService ijs;

    /**
     * 测试本地服务
     */
    @Test
    public void testServiceFromLocalContext() throws JobServiceException {
        try {
            int returnVal = ijs.updateTestStatus(1L);
            Assert.assertTrue(returnVal == 1);
        } catch (JobServiceException ex) {
            ex.printStackTrace();
        }
    }
}
```

### 结果

发现最后打印出来的log为: 

```
2017-01-16 14:04:23.810 INFO me.ele.fin.job.service.JobService[main]: [unknown 1.1 unknown^^2337080905338436383|1484546598112] ## update result: 0 
2017-01-16 14:04:23.810 INFO me.ele.fin.job.service.JobService[main]: [unknown 1.1 unknown^^2337080905338436383|1484546598112] ## Duration milliseconds: 51288 
```

同时字段status的值并没有变更

### 结论

- `@Transactional(timeout = ?)`表示超时多长时间后，应用逻辑中mysql事务失败，事务会rollback，但是对应的sql已经发出，依然会争抢行锁。并且`@Transactional(timeout = ?)`不能控制在timeout时间后结果返回client，需要等到`innodb_lock_wait_timeout`设置的时间后，client才能得到返回结果
