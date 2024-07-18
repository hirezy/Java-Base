# 前言
>文本已收录至我的GitHub仓库，欢迎Star：https://github.com/bin392328206/six-finger                             
> **种一棵树最好的时间是十年前，其次是现在**   
>我知道很多人不玩**qq**了,但是怀旧一下,欢迎加入六脉神剑Java菜鸟学习群，群聊号码：**549684836** 鼓励大家在技术的路上写博客
![](https://user-gold-cdn.xitu.io/2019/11/30/16ebbfc45ff83dba?w=640&h=496&f=jpeg&s=28818)

## 絮叨 
> 自在地境 已经在武侠的世界里算的上一放强者了 所以呢基础的东西今天就不说了，       
> 如果要去看基础的话建议看我下面的链接：  
>[🔥从零开始学Redis之金刚凡境](https://juejin.im/post/5dde62bf5188256ebc1ee256)  
>第一篇基础的概率很多 估计大家看得想睡觉 那么这一篇    
> 我保证全是满满的干货，不管是工作还是面试 都会让人眼前一亮，让人觉得这家伙 **有点意思**


## 缓存雪崩、击穿、穿透、缓存一致性问题
> 看到这大家可能会想 这博主是不是sha，这东西基本上背过面试题的，谁特么不会呀。。。

> 博主我也承认这是老生常谈的东西，但是也是很重要的东西，今天就带大家完整的解决这些问题

> 先来看看缓存的处理流程：

![](https://user-gold-cdn.xitu.io/2019/12/1/16ebff3ab806eda2?w=447&h=308&f=png&s=26503)

>  前台请求，后台先从缓存中取数据，取到直接返回结果，取不到时从数据库中取，数据库取到更新缓存，并返回结果，数据库也没取到，那直接返回空结果。
## 缓存雪崩
### 什么是缓存雪崩呢 
> 就是数据未加载到缓存中，或者缓存同一时间大面积的失效，从而导致所有请求都去查数据库，导致数据库CPU和内存负载过高，甚至宕机。假设一万个请求同时打到你的数据库，那么你的数据库不用想肯定给搞死了，这就是缓存雪崩

### 一个缓存雪崩发过程
1.  redis集群大面积故障
2.  缓存失效，但依然大量请求访问缓存服务redis
3.  redis大量失效后，大量请求转向到mysql数据库
4.  mysql的调用量暴增，很快就扛不住了，甚至直接宕机
5.  由于大量的应用服务依赖mysql和redis的服务，这个时候很快会演变成各服务器集群的雪崩，最后网站彻底崩溃。

> 由此可见 缓存雪崩的情况出现对于一个产品来说，可能会让用户对它失去信心，所以防止缓存雪崩是非常有必要的

### **如何解决缓存雪崩**
- 第一种方案： 缓存层设计成高可用，防止缓存大面积故障。即使个别节点、个别机器、甚至是机房宕掉，依然可以提供服务，例如 Redis Sentinel 和 Redis Cluster 都实现了高可用。

- 第二种方案：在批量往Redis存数据的时候，把每个Key的失效时间都加个随机值就好了，这样可以保证数据不会在同一时间大面积失效，我相信，Redis这点流量还是顶得住的。

```
redisUtils.set(RedisKeys.App.appApiFrontHomePageOrg(),homePageOrgDTOList,RedisUtils.FRONT_END_EXPIRE+RandomUtil.randomInt(1,100)*100);
```
> 最后的随机数就是防止首页缓存同时失效的

- 加互斥锁。当发现没有命中Redis，去查数据库的时候，在执行更新缓存的操作上加锁，谁拿到锁谁去更新，同时在拿到锁之后先从缓存再获取一次如果有就返回，没有就查库然后更新(这种方案其实也是可以的)


```

<!-- redis所需 (自带lettuce 5.1.4.RELEASE) lettuce是springboot2.x后连接redis推荐使用的客户端 -->
<dependency>
   <groupId>org.springframework.boot</groupId>
   <artifactId>spring-boot-starter-data-redis</artifactId>
</dependency>
```

> 这个是具体的方法（业务代码）

```
 public void testRedis(){
        String key="六脉神剑";
        // 通过key获取value
        String value = (String) redisTemplate.opsForValue().get(key);
        if (StringUtil.isEmpty(value)) {
            String lockKey="一剑断落水";
            try {
                boolean locked = RedissLockUtil.tryLock(lockKey, TimeUnit.SECONDS, 5, 10);
                if (locked) {
                   System.out.println("我是获得锁的线程，求叫我大佬"+Thread.currentThread().getName());
                    User byId = service.getById(1);
                    System.out.println("睡一下,然后把从数据库查到的数据放到redis中,生产环境中没必要睡，我这个是测试");
                    Thread.sleep(500);
                    redisTemplate.opsForValue().set(key,"byId");
                    redisTemplate.delete(lockKey);

                } else {
                    // 其它线程进来了没获取到锁便等待50ms后重试
                    System.out.println("我们是没有得到锁的线程，等待一下再试试");
                    Thread.sleep(50);
                    testRedis();
                }
            } catch (Exception e) {

            } finally {
                redisTemplate.delete(lockKey);
            }
        }

    }
```
下面是测试方法 和测试结果

```
    @Test
    public void testCountDownLatch() {

        int threadCount = 10;

        final CountDownLatch latch = new CountDownLatch(threadCount);

        for (int i = 0; i < threadCount; i++) {

            new Thread(new Runnable() {

                @Override
                public void run() {

                    System.out.println("线程" + Thread.currentThread().getId() + "开始出发");

                    try {
                        userController.testRedis();

                    } catch (Exception e) {
                        e.printStackTrace();
                    }

                    System.out.println("线程" + Thread.currentThread().getId() + "已到达终点");

                    latch.countDown();
                }
            }).start();
        }

        try {
            latch.await();
        } catch (InterruptedException e) {
            e.printStackTrace();
        }

        System.out.println("10个线程已经执行完毕！开始计算排名");
    }
```


```
线程47开始出发
线程49开始出发
线程48开始出发
线程50开始出发
线程52开始出发
线程51开始出发
线程53开始出发
线程55开始出发
线程54开始出发
线程56开始出发
2019-12-01 11:42:34.757  INFO 222916 --- [       Thread-3] io.lettuce.core.EpollProvider            : Starting without optional epoll library
2019-12-01 11:42:34.760  INFO 222916 --- [       Thread-3] io.lettuce.core.KqueueProvider           : Starting without optional kqueue library
我是获得锁的线程，求叫我大佬Thread-2
睡一下,然后把从数据库查到的数据放到redis中,生产环境中没必要睡，我这个是测试
我们是没有得到锁的线程，等待一下再试试
我们是没有得到锁的线程，等待一下再试试
我们是没有得到锁的线程，等待一下再试试
我们是没有得到锁的线程，等待一下再试试
我们是没有得到锁的线程，等待一下再试试
我们是没有得到锁的线程，等待一下再试试
我们是没有得到锁的线程，等待一下再试试
我们是没有得到锁的线程，等待一下再试试
我们是没有得到锁的线程，等待一下再试试
线程47已到达终点
我是获得锁的线程，求叫我大佬Thread-7
睡一下,然后把从数据库查到的数据放到redis中,生产环境中没必要睡，我这个是测试
我们是没有得到锁的线程，等待一下再试试
我们是没有得到锁的线程，等待一下再试试
我们是没有得到锁的线程，等待一下再试试
我们是没有得到锁的线程，等待一下再试试
我们是没有得到锁的线程，等待一下再试试
我们是没有得到锁的线程，等待一下再试试
我们是没有得到锁的线程，等待一下再试试
我们是没有得到锁的线程，等待一下再试试
线程55已到达终点
线程50已到达终点
线程53已到达终点
线程49已到达终点
线程51已到达终点
线程54已到达终点
线程48已到达终点
线程56已到达终点
线程52已到达终点
10个线程已经执行完毕！开始计算排名


```
> 从以上测试可以看出 一开始10个并发 只有线程47拿到了锁， 然后 其他线程一直再重新获得锁 然后等二轮的时候线程55拿到锁 ，之后的就是全部拿的缓存了 ，第二个能拿到锁 是因为这个是可重入锁。  
然后我测了100个并发也是发现最多只有2个线程去查数据库， 所以这种方式可以解决缓存雪崩的问题

## 缓存击穿

### 什么是缓存击穿
我个人理解 击穿 就是正面刚 比如我是矛 你是盾 我直接把你的盾击穿， 就是比如 几个热点Key 同时几百万并发直接把redis 干掉了， 然后数据全部打到数据库的情况，或者是redis的这几个热点数据失效的情景下，同时全部的并发查这个热数据，导致最后打到数据库的情况 这个就是缓存击穿。

###　**如何解决缓存击穿**
- 还是分布式锁 哈哈 因为分布式锁能控制到数据库的最后一到防线 
- redis做集群 哨兵 （下一篇讲）


## 缓存穿透
### 什么是缓存穿透
>  缓存穿透是指缓存和数据库中都没有的数据，而用户不断发起请求，如发起为id为“-1”的数据或id为特别大不存在的数据。这时的用户很可能是攻击者，攻击会导致数据库压力过大。  

> **这个和缓存击穿不一样,缓存击穿人家好歹有个盾,你这就是直接跟人家肉搏干。啥也没有**

###　**如何解决缓存穿透**

1.  第一种方案 和上面的双重锁一样 如果是拿到数据库为空 那么就给这个key 设置一个null值 时间设置短一点 30s， 这样下次并发进来就不会说把数据打到我们的数据库上了
```
redisTemplate.opsForValue().set(key,"byId",30,TimeUnit.SECONDS);
```
2. 就是我们写代码的时候 要对一些非法的请求参数校验 我相信大家都是这样做的。

3. 第二种方案 采用我们第一篇中学到的一个高级用法 bitMap([不懂的可以翻我前面一篇文章](https://juejin.im/post/5dde62bf5188256ebc1ee256))
> 下面是具体的代码

```
  public void testRedis() {
        List<String> goodsId = new ArrayList<String>();
        goodsId.add("201912010001");
        goodsId.add("20191201002");
        goodsId.add("20191201003");
        goodsId.add("20191201004");

        String key = "六脉神剑";
        // 通过key获取value
        long index = Math.abs((long) (goodsId.get(3).hashCode()));
        Boolean goodsIdBit = redisTemplate.opsForValue().getBit("goodsIdBit",index);
        if (!goodsIdBit) {
            return;
        }
        String value = (String) redisTemplate.opsForValue().get(key);
        if (StringUtil.isEmpty(value)) {
            String lockKey = "一剑断落水";
            try {
                /**
                 * 尝试获取锁
                 * @param lockKey
                 * @param unit 时间单位
                 * @param waitTime 最多等待时间
                 * @param leaseTime 上锁后自动释放锁时间
                 * @return
                 */
                boolean locked = RedissLockUtil.tryLock(lockKey, TimeUnit.SECONDS, 5, 10);
                if (locked) {
                    System.out.println("我是获得锁的线程，求叫我大佬" + Thread.currentThread().getName());
                    User byId = service.getById(1);
                    System.out.println("睡一下,然后把从数据库查到的数据放到redis中,生产环境中没必要睡，我这个是测试");
                    Thread.sleep(5000);
                    redisTemplate.opsForValue().set(key, "byId", 30, TimeUnit.SECONDS);
                    //设置bitMap
                    redisTemplate.opsForValue().setBit("goodsIdBit", index, true);
                    redisTemplate.delete(lockKey);

                } else {
                    // 其它线程进来了没获取到锁便等待50ms后重试
                    System.out.println("我们是没有得到锁的线程，等待一下再试试");
                    Thread.sleep(50);
                    testRedis();
                }
            } catch (Exception e) {

            } finally {
                redisTemplate.delete(lockKey);
            }
        }

    }
```
一个简单结论

```
   List<String> goodsId = new ArrayList<String>();
        goodsId.add("201912010001");
        goodsId.add("20191201002");
        goodsId.add("20191201003");
        goodsId.add("20191201004");

        long index = Math.abs((long) (goodsId.get(3).hashCode()));
        System.out.println(index);
        redisTemplate.opsForValue().setBit("goodsIdBit",index,true);
        Boolean aBoolean = redisTemplate.opsForValue().getBit("goodsIdBit", index);
        System.out.println(aBoolean);
        
        590648468
2019-12-01 14:48:34.099  INFO 220368 --- [           main] io.lettuce.core.EpollProvider            : Starting without optional epoll library
2019-12-01 14:48:34.100  INFO 220368 --- [           main] io.lettuce.core.KqueueProvider           : Starting without optional kqueue library
true

```
> 其实就是和前面的锁差不多 多了一点步骤 就是先看bitmap中有没有这个商品id 有才让它去redis中查商品id的数据 然后保存的时候也要setbit这个商品 id

> 其实我这个写法很垃圾 但是我不知道怎么实现去实现一个布隆过滤器 那让估计好点 有大神可以指点一下，不胜感激


## 缓存一致性问题

### 什么是缓存一致性问题
> 这个简单 就是我们数据库的数据和缓存中的数据不一致，这就是缓存不一致

### 几种方式缓存不一致的原因和解决方案

- 方案一 **先更新数据库，再删缓存**

> 这个方案的问题是什么呢？ 就是假设我们更新数据成功了 然后去删除缓存的时候失败了 这就导致了缓存中是老数据，会造成缓存不一致

> 那么我这边提供几种简单的解决方案   

> 第一个就是 我们删除的时候 把删除缓存的操作做个5次的循环 ，如果5次删除操作都失败， 我们就把这个动作用mq发送出去用消费删除

> 第二个就是用一个中间件**canal** 去兼听mysql的**binlog** 然后 从binlong中解析出要删除的字段 然后 继续上面第一个的方式（这个方式的好处 全程也算是异步的跟业务代码是没有关系的）我把流程图贴下：

![](https://user-gold-cdn.xitu.io/2019/12/1/16ec0db1f4083f22?w=743&h=505&f=png&s=52288)

- 方案二 **先更新数据库，再更新缓存** 
> 这个操作 问题更多感觉 首先 更新数据成功 更新缓存失败，或者是开始更新数据库成功 然后更新缓存成功 然后事务回滚，也是缓存不一致。

- 方案三 **删除缓存 再更新数据库**
>看起来好像最好 我反正是删除缓存了 就算更新失败 下次去读也是最新的数据（一切看起来很美好）,

>其实不然，试想2个并发一个更新 一个查询 你先更新的时候 删除了缓存 但是此时 查询发现没有缓存 然后吧数据缓存到了数据库 就会去查数据库 但是此时更新的又更新成功，最后就会再很长的一个时间内 缓存和数据库是不一致的，所以这种是方案是不可取的

**综上所诉，我觉得最好的方式先查再删除 然后再配合订阅binlong 来做多重删除的方式是不错的，可能我接触的不是很多，希望各位大佬有更好的方式提出**

## 结尾
>redis的几个比较棘手的问题就讲到这 后面还有 **lua脚本 主从 哨兵** 等我们下集再见

> 因为博主也是一个开发萌新 我也是一边学一边写 我有个目标就是一周 二到三篇 希望能坚持个一年吧 希望各位大佬多提意见，让我多学习，一起进步。
## 日常求赞
> 好了各位，以上就是这篇文章的全部内容了，能看到这里的人呀，都是**人才**。

> 创作不易，各位的支持和认可，就是我创作的最大动力，我们下篇文章见

>六脉神剑 | 文 【原创】如果本篇博客有任何错误，请批评指教，不胜感激 ！
