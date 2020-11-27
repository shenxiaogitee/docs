### RabbitMQ延时队列

**场景:**

​	比如未付款订单,超过一定时间后,系统自动取消订单并释放占有物品。

![](https://gitee.com/enioy/img/raw/master/K8S/20201126092435.png) 

**常用解决方案:**

​	spring的schedule定时任务轮询数据库

**缺点:**

​	消耗系统內存、增加了数据库的压力、存在较大的时间误差

![](https://gitee.com/enioy/img/raw/master/K8S/20201126092621.png) 

​	本来是30分钟未支付关闭订单，实际第60分钟的定时任务扫描才扫到，此时实际超时59分钟了。

**解决:** 

​	RabbitMQ的**消息TTL**和**死信Exchange**结合



#### 消息的TTL( Time To live)
- 消息的TTL就是**消息的存活时间**

- RabbitMQ可以对**队列**和**消息**分别设置TTL

  - 对队列设置就是队列没有消费者连着的保留时间,**也可以对毎一个单独的消息做单独的设置,超过了这个时间,我们认为这个消息就死了,称之为死信**。

  - 如果队列设置了,消息也设置了,那么会**取小**的。所以一个消息如果被路由到不同的队列中,这个消息死亡的时间有可能不一样(不同的队列设置)。这里单讲单个消息的TTL,因为它才是实现延迟任务的关键。可以通过**设置消息的expiration字段或者x-message-ttl属性来设置时间**,两者是一样的效果 



#### Dead Letter Exchanges(DLX)
- 一个消息在满足如下条件下,**会进死信路由**,记住这里是路由而不是队列,一个路由可以对应很多队列。

  - 一个消息被Consume拒收了,并且reject方法的参数里requeue是false。也就是说不会被再次放在队列里,被其他消费者使用。(basic.reject/ basic.nack) requeue=false

  - 上面的消息的TTL到了,消息过期了

  - 队列的长度限制满了。排在前面的消息会被丢弃或者扔到死信路由上

- Dead Letter Exchange其实就是一种普通的exchange,和创建其他exchange没有两样。只是在某一个设置 Dead Letter Exchange的队列中有消息过期了,会自动触发消息的转发,发送到 Dead Letter Exchange中去。

- 我们既可以控制消息在一段时间后变成死信,又可以控制变成死信的消息被路由到某一个指定的交换机,结合二者,其实就可以实现一个延时队列

  

#### 使用消息TTL+死信路由实现延时队列

 

![](https://gitee.com/enioy/img/raw/master/K8S/20201126103136.png)  

![](https://gitee.com/enioy/img/raw/master/K8S/20201126103508.png) 



不建议使用消息设置TTL来实现延时队列！推荐使用队列设置过期时间

**原因：** RabbitMq采用惰性检查机制，也就是懒检查机制，比如消息队列中存放了多条消息，第一条是5分钟过期，第二条是1分钟过期，第三条是1秒钟过期，按照正常的过期逻辑，应该是1秒过期的先排出这个队列，进入死信队列中，但是实际RabbitMQ是先拿第一条消息，也就是5分钟过期的，一看5分钟还没到过期时间，然后等待5分钟会将第一条消息拿出来，放入死信队列，这里就会出现问题，第二条设置1分钟的和第三条设置1秒钟的消息必须要等待第一条5分钟过期后才能过期，等待第一条消息过期5分钟了，拿第二条、三条的时候都不需要判断就已经过期了，直接就放入死信队列中，所以第二条、三条需要等待第一条消息过了5分钟才能过期，这样的延时根本就没产生对应的效果！



![](https://gitee.com/enioy/img/raw/master/K8S/20201126104811.png) 



![](https://gitee.com/enioy/img/raw/master/K8S/20201126112649.png) 



![](https://gitee.com/enioy/img/raw/master/K8S/20201126113256.png) 

  

```java
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-amqp</artifactId>
</dependency>
```

```java
spring:
  rabbitmq:
    host: 192.168.1.110
    port: 5672
    virtual-host: /
```

 

```java
package com.atguigu.gulimall.ware.config;

import org.springframework.amqp.core.Binding;
import org.springframework.amqp.core.Exchange;
import org.springframework.amqp.core.Queue;
import org.springframework.amqp.core.TopicExchange;
import org.springframework.amqp.rabbit.annotation.EnableRabbit;
import org.springframework.amqp.rabbit.annotation.RabbitListener;
import org.springframework.amqp.support.converter.Jackson2JsonMessageConverter;
import org.springframework.amqp.support.converter.MessageConverter;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

import java.util.HashMap;

/**
 * RabbitMQ配置
 */
@EnableRabbit
@Configuration
public class MyRabbitConfig {

    public static final String STOCK_EVENT_EXCHANGE = "stock.event.exchange";
    public static final String STOCK_DELAY_QUEUE = "stock.delay.queue";
    public static final String STOCK_RELEASE_STOCK_QUEUE = "stock.release.stock.queue";
    public static final String STOCK_LOCKED_ROUTING_KEY = "stock.locked";
    public static final String STOCK_RELEASE_ROUTING_KEY = "stock.release.#";

    /**
     * 连上RabbitMQ
     */
    @RabbitListener(queues = MyRabbitConfig.STOCK_DELAY_QUEUE)
    public void listener() {

    }

    /**
     * 使用JSON序列号机制进行消息转换
     */
    @Bean
    public MessageConverter messageConverter() {
        return new Jackson2JsonMessageConverter();
    }

    /**
     * @Bean作用：
     * 当我们连上RabbitMQ后，容器中的 Queue Exchange Binding会自动创建(RabbitMQ中没有的情况下)
     */
    @Bean
    public Exchange stockEventExchange() {
        return new TopicExchange(MyRabbitConfig.STOCK_EVENT_EXCHANGE, true, false);
    }

    @Bean
    public Queue stockDelayQueue() {
        HashMap<String, Object> arguments = new HashMap<>();
        arguments.put("x-dead-letter-exchange", MyRabbitConfig.STOCK_EVENT_EXCHANGE);
        arguments.put("x-dead-letter-routing-key", MyRabbitConfig.STOCK_RELEASE_ROUTING_KEY);
        arguments.put("x-message-ttl", 120000);
        return new Queue(MyRabbitConfig.STOCK_DELAY_QUEUE, true, false, false, arguments);
    }

    @Bean
    public Queue stockReleaseStockQueue() {
        return new Queue(MyRabbitConfig.STOCK_RELEASE_STOCK_QUEUE, true, false, false);
    }

    @Bean
    public Binding stockLockedBinding() {
        return new Binding(MyRabbitConfig.STOCK_DELAY_QUEUE,
                Binding.DestinationType.QUEUE,
                MyRabbitConfig.STOCK_EVENT_EXCHANGE,
                MyRabbitConfig.STOCK_LOCKED_ROUTING_KEY,
                null);
    }

    @Bean
    public Binding stockReleaseBinding() {
        return new Binding(MyRabbitConfig.STOCK_RELEASE_STOCK_QUEUE,
                Binding.DestinationType.QUEUE,
                MyRabbitConfig.STOCK_EVENT_EXCHANGE,
                MyRabbitConfig.STOCK_RELEASE_ROUTING_KEY,
                null);
    }

}

```

  

#### 库存解锁的场景
1. 下订单成功,订单过期没有支付被系统自动取消、被用户手动取消。都要解锁库存
2. 下订单成功,库存锁定成功,接下来的业务调用失败,导致订单回滚,之前锁定的库存就要自动解锁。