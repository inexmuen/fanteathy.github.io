---
title: 关于timeout
date: 2017-01-16 15:00:00
author: Fanteathy
tags: arch
categories: arch
---

## 使用

`Timeout`，是一种常见的容错模式。常见的有设置网络连接超时时间，一次RPC的响应超时时间等。在分布式服务调用的场景中，它主要解决了当依赖服务出现建立网络连接或响应延迟，不用无限等待的问题，调用方可以根据事先设计的超时时间中断调用，及时释放关键资源，如Web容器的连接数，数据库连接数等，避免整个系统资源耗尽出现拒绝对外提供服务这种情况。

意义: 

- 当网络出现短暂的抖动，或者下游服务调用超时时，合理的超时时间可以避免连接池快速被占满，可有效延长服务的可用时长
- 可以结合降级，熔断等SOA治理的手段使用

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

### zeus_core中的超时

#### 1. api_hard_timeout为接口的真实超时设置，为server的超时设置。是gevent执行对应协程时的Timeout，为server应用逻辑的timeout

***设置***:

```
service = Service(
    name=NAME,
    slug=NAME[NAME.find('.')+1:],
    timeout=3 * 1000,
    api_hard_timeout=SERVICE_API_HARD_TIMEOUT,
    ...
```

***实现***:

```
with gevent.Timeout(hard_timeout):
    try:
        result = func(dispatcher, *args)
        ...
```

#### 2. timeout为软超时，目前只是打个warning,发个signal(无实际价值)

#### 3. 调用其他服务时候的超时，为client的超时设置。是网络请求时的timeout

***设置***:

```
'payment': {
            'pool': 'huskar',
            'name': 'me.ele.payment.service',
            'client': 'http',
            'timeout': config.get('config:soa_client_timeout:payment', 1),
            'cluster': config.get('config:cluster:payment', "stable"),
            'iface': 'me.ele.payment.api.service.PaymentService',
            'thrift_file': path.join(
                current_path, 'thrift_files/payment/PaymentService.thrift'),
        }

```

表示`EOS`调用`payment`服务时的eos超时时间

***实现***:(参考zeus_client使用http client中的代码实现)

```
from thriftpy.transport import TSocket, TBufferedTransport
from . import Client, THTTPJsonProtocol

socket = TSocket(host, port)
socket.set_timeout(timeout)
...
```

### Pylon对应的超时

```
private Object callCommand(Task task, BreakerMetrics metric) throws Throwable {
        Future<?> result = null;
        try {
            result = service.submit(task);
            long timeout = task.getTimeoutInMillis();
            Object ret = result.get(timeout > 0 ? timeout : property.getTimeout(), TimeUnit.MILLISECONDS);
            metric.increment(BreakerStatus.SUCCESS);
            if (metric.inTestPhase()) {
                metric.singleTestPass(true);
            }
            return ret;
        } catch (InterruptedException | TimeoutException e) {
            metric.increment(BreakerStatus.TIMEOUT);
            if (result != null) {
                result.cancel(true);
            }
            task.cancel();
            task.setStatus(CallStatus.timeout);
            throw new RequestTimeoutException(String.format("Service(%s) occurs a request execution timeout!", property.getService()));
...

```

Java的超时设置可以参考:

[Java: Set timeout for threads in a ThreadPool](http://stackoverflow.com/questions/14236654/java-set-timeout-for-threads-in-a-threadpool)

对应代码如下:

```
Future<?> future = null;

for (List<String> l : partition) {
    Runnable worker = new WorkerThread(l);
    future = executor.submit(worker);
}

try {
    System.out.println("Started..");
    System.out.println(future.get(3, TimeUnit.SECONDS));
    System.out.println("Finished!");
} catch (TimeoutException e) {
    System.out.println("Terminated!");
}
```

## 最佳实践

为了让client尽快得到响应，也为了尽量减少服务响应延迟时的服务资源消耗，需要设置`client timeout(客户端timeout)`, `server timeout(服务端timeout)`以及`service timeout(基础服务如mysql等timeout)`。同时需要根据实际情况设置一条请求链路上对应client,server和service的timeout。

一般需要考虑以下情况:

- 设置client timeout保证client尽快得到响应，同时方便重试
- 设置server timeout保证及时释放服务器资源，保证不被client拖垮
- 设置一些基础服务的timeout，可以有效保护对应资源，避免整个系统资源耗尽出现拒绝对外提供服务这种情况
- timeout值的考虑
	- 一条链路上游至下游如果只是一次Request/Response的话，那么对应的timeout值应该逐渐减少，如RPC请求的Client和Server端（保证请求的时间范围覆盖了响应的时间范围）
	- 如果有多次交互的话，无需要考虑。如server请求DB，Server先发送commit到DB，若Server超时后再发rollback给DB。则DB的timeout无需要比Server短，只需要考虑DB自身的性能即可

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
    @Transactional(timeout = 10)
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

### 结果及结论

使用postman模拟client调用对应接口，设`select for update`为事务1，接口调用为事务2。

- `0 - 10s` 内，提交事务1(运行`commit`手动提交), 则事务2可以获取innodb锁往下执行，最后事务2也正确提交
- `10 - 50s` 内，提交事务1，则事务2`roll back`，同时立刻返回结果给client，表示`@Transactional(timeout = ?)`可以设置在多长时间后，应用逻辑中mysql事务失败，事务2会rollback，同时返回结果给client（快速失败）
- `>50s` 后，由于`innodb_lock_wait_timeout`的存在，事务1和事务2都会roll back，且打印以下log

```
2017-01-16 14:04:23.810 INFO me.ele.fin.job.service.JobService[main]: [unknown 1.1 unknown^^2337080905338436383|1484546598112] ## update result: 0 
2017-01-16 14:04:23.810 INFO me.ele.fin.job.service.JobService[main]: [unknown 1.1 unknown^^2337080905338436383|1484546598112] ## Duration milliseconds: 51288 
```