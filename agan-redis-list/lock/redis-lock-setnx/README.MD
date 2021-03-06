
### 1.为什么要使用分布式锁，它解决了什么问题？
锁，顾名思义，就是一份数据在同一时间内只能被一个人使用，不能2个人同时使用。
对于锁，一般有2种使用场景：
- 单机系统：单机系统在多用户多线程并发操作同一份资源（数据）的时候，采用线程加锁的机制，
即当某个线程获取到该资源（数据）后，立即加锁，当使用完后，再解锁，其它线程就可以接着使用了。
例如，在JAVA的锁机制synchronize/Lock等。

- 分布式系统：在分布式系统环境中，单机的线程锁机制是不起作用的，因为系统采用了集群部署在不同的机器上;
如果是多用户同时访问同一份资源（数据）时，JAVA处理锁机制的synchronize/Lock是起不到作用的，
因为资源（数据）在不同的服务器之间共享。

因此，针对分布式环境，我们必须采用分布式锁。
分布式锁解决2个问题：
1. 解决多用户操作的幂等性问题
典型案例：用户下订单操作，为了避免用户重复提交导致出现多条相同订单，一般采用分布式锁来解决订单的幂等性问题。
2. 把多用户的并行操作转化为串行操作
典型案例：在秒杀系统的高并发减库存操作，例如 库存只剩下3台手机，A用户秒杀2台，B用户也秒杀2台，如果A、B用户同时下单就会出现库存等于-1的情况。

### 2.分布式锁必须具备5个特性
设计一个好的分布式锁，一般要具备以下4个特性：
1. 互斥性：指不可能同时2个人(线程)以上的人拿到锁。
2. 可用性：redis集群环境下，不能因为某个节点瘫痪，导致客户端不能获取和释放锁。
3. 终止性：为了避免死锁，必须有自动的终止或撤销锁操作，一般是采用超时处理机制。
4. 抢占性：别人（其他线程）已经占了锁，不能私下解锁别人（其他线程）的锁，必须等等锁的释放
5. 可重入性: 同一个节点上的同一个线程如果获取了锁之后那么也可以再次获取这个锁。

### 3.业界有哪些主流的分布式锁实现方案？
目前主流的有三种，如下：
- 基于数据库实现
基于数据库来做分布式锁的话，通常是采用数据库的乐观锁或悲观锁来实现。
- 基于ZooKeeper实现
基于ZooKeeper，是采用它的临时有序节点来实现的分布式锁。
- 基于Redis实现
基于Redis实现的锁机制，主要是依赖redis自身的原子来实现
以上三种方式都可以实现分布式锁，如果并发量不大的话，直接采用数据库就可以，如果高并发的话，就要考虑zookeeper或redis,但从高并发高性能角度考虑，基于 Redis 实现性能会更好；所以如何选择，还是取决于业务需求。





### 4.基于Redis的SETNX实现分布式锁
- 步骤1：采用SETNX加锁
采用SETNX 来实现，NX 是not exist 的意思。
例如：
```cmd
127.0.0.1:6379> setnx lock01 true
(integer) 1
127.0.0.1:6379> setnx lock01 true
(integer) 0
```
如果 SETNX 返回1，说明拿到锁了。
如果 SETNX 返回0，说明拿锁失败，被其他线程占用。

- 步骤2：为了避免死锁，增加超时机制
为该锁增加10分钟超时，防止死锁
```cmd
127.0.0.1:6379> expire lock01 600
(integer) 1
```
以上操作有一个问题?
无法保证锁的原子性操作，因为要操作2个步骤 setnx   expire
用以下来代替：

SET key value [NX] [XX] [EX <seconds>] [PX [millseconds]] 设置一对key value
必选参数说明  
SET：命令
key：待设置的key
value: 设置的key的value

可选参数说明  
NX：表示key不存在才设置，如果存在则返回NULL
XX：表示key存在时才设置，如果不存在则返回NULL
EX seconds：设置过期时间，过期时间精确为秒
PX millsecond：设置过期时间，过期时间精确为毫秒


以上set 代替了 setnx +expire 需要分2次执行命令操作的方式，保证了原子性
``` cmd
127.0.0.1:6379> set lock true NX px 600000
OK
127.0.0.1:6379> set lock true NX px 600000
(nil)
```
如果setnx 返回ok 说明拿到了锁
如果setnx 返回 nil，说明拿锁失败，被其他线程占用。








### 5.案例实战：基于redis的分布式锁实现下订单防止重复提交

先来说下这种业务场景：
用户提交订单的时候，如果客户端APP或JS没做校验的话，用户对订单的提交按钮重复点击，就会造成数据库出现多笔订单。
为了避免用户重复提交导致数据库出现多条相同订单，一般采用分布式锁来解决订单的幂等性问题。


``` java
@PostMapping(value = "/createOrder", produces = APPLICATION_JSON_UTF8_VALUE, consumes = APPLICATION_JSON_UTF8_VALUE)
    public String createOrder(@RequestBody  OrderDTO obj) {

        //步骤1：先转换为唯一MD5
        String json=JsonUtil.object2Json(obj);
        String md5 = DigestUtils.md5DigestAsHex(json.getBytes()).toUpperCase();

        //步骤2：把md5设置为分布式锁的key
        /**
         * setIfAbsent 的作用就相当于 SET key value [NX] [XX] [EX <seconds>] [PX [millseconds]]
         * 设置 5分钟过期
         */
        Boolean bo=stringRedisTemplate.opsForValue().setIfAbsent(md5,"1",60*5, TimeUnit.SECONDS);
        if(bo){
            // 加锁成功
            log.debug("{}拿锁成功，开始处理业务",md5);
            try {
                //模拟10秒 业务处理
                Thread.sleep(1000*10);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            //步骤3：解锁
            stringRedisTemplate.delete(md5);
            log.debug("{}拿锁成功，结束处理业务",md5);
            return "ok";
        }else{
            log.debug("{}拿锁失败",md5);
            //拿不锁，直接退出
            return  "请不要重复点击！";
        }
    }

```










