# 秒杀系统

## 项目描述

该项目是我自己设计的一个秒杀系统后台。项目主要有以下几个业务：

- 用户登录
- 购买商品
- 生成订单

其中用户登录功能采用的是分布式登陆，将用户信息保存到了 cookie和 redis 中。

## 项目实现

### 1. 判断用户是否登录

从 request 中获取 cookie，然后根据 cookie 中的值去获取用户数据。若获取成功，则用户已登录，否则未登录。

### 2. 判断商品是否已经卖完了

- version1：查询数据库中的商品库存

- version2：将商品库存保存在 redis 中，并且和数据库中的库存同步。

- version3：通过使用 ConcurrentHashMap，key 为商品ID，value 为布尔值，标记某商品是否卖完，后面只要通过查询这个 Map 中的 key 值，就可以判断了。

### 3. 判断用户是否重复秒杀同一商品

- version1：查询订单数据库是否存在
- version2：当用户秒杀商品成功后，会在 redis 中存储该信息。然后只要判断是否能从 redis 中取出该值就可以判断了。

### 4. 预减库存

- 项目启动时，对库存进行预热，将每个商品的库存信息保存在 redis 中，然后在 redis 中预减库存

### 5. 创建秒杀信息并放入消息队列中

通过消息队列，实现流量削峰。秒杀开始瞬间，流量很大，通过使用消息队列的方式，可以实现异步下单。

### ~~6. 判断库存是否充足~~

~~从数据库中获取商品库存，已经优化为第三步的预减库存。~~

### 7. 扣减商品库存

扣减数据库中的商品库存

### 8. 生成订单

生成订单并插入数据库中。同时根据用户ID与商品ID建立唯一索引，确保用户不会重复秒杀同一商品。

### 9. 将秒杀成功的用户ID与商品ID保存到redis中，用于第二步的判断

### 10. 错误处理

在处理秒杀消息的过程中，任一过程出现问题都会抛出异常，并被捕获，将预减的库存加回来，同时判断库存是否大于0来确定 ConcurrentHashMap 中的商品是否卖完。

### 11. 兜底方案之限流

通过令牌桶限流的方式，进一步降低并发。QPS 为下降很多，不得已不要用。

## 项目测试

1. 为了模拟高并发的用户请求，这里先将登录功能给注释掉，用随机数字来模拟用户ID。

```java
@PostMapping("/goods")
public String seckillGoods(@RequestParam("goodsId") Integer goodsId) {
    // 注释掉用户登录功能
    // 判断用户是否登录
    //        SkUser skUser = isLogin();
    //        if (skUser == null) {
    //            log.info("用户未登录！");
    //            return "login";
    //        }
    // 使用随机字数字模拟 useId
    int userId = new Random().nextInt(4000);
}
```



2. 使用 Jmeter 进行压力测试

创建多个线程，POST 请求该地址
http://localhost:8080/seckill/goods