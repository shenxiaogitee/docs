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



![](https://gitee.com/enioy/img/raw/master/K8S/20201127160521.png)  

  

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



![](https://gitee.com/enioy/img/raw/master/K8S/20201127152350.png) 





#### Spring Boot中使用延时队列
1. Queue、 Exchange、 Binding可以@Bean进去

2. 监听消息的方法可以有三种参数(不分数量,顺序)

   Object content, Message message, Channel channel

3. channel可以用来拒绝消息,否则自动ack;



#### 如何保证消息可靠性

##### 1.消息丢失

- 消息发送出去,由于网络问题没有抵达服务器

  - 做好容错方法(try- catch),发送消息可能会网络失败,失败后要有重试机制,可记录到数据库,采用定期扫描重发的方式

  - 做好日志记录,每个消息状态是否都被服务器收到都应该记录

  - 做好定期重发,如果消息没有发送成功,定期去数据库扫描未成功的消息进行重发

  - ```sql
    CREATE TABLE `mq_message` (
      `message_id` char(32) NOT NULL,
      `content` text,
      `to_exchane` varchar(255) DEFAULT NULL,
      `routing_key` varchar(255) DEFAULT NULL,
      `class_type` varchar(255) DEFAULT NULL,
      `message_status` int(1) DEFAULT '0' COMMENT '0-新建 1-已发送 2-错误抵达 3-已抵达',
      `create_time` datetime DEFAULT NULL,
      `update_time` datetime DEFAULT NULL,
      PRIMARY KEY (`message_id`)
    ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
    ```

    ```java
    // TODO 保证消息一定会发送出去，每一个消息都做好日志记录，定期扫描数据库将失败的消息再发送一遍
    try {
        rabbitTemplate.convertAndSend(MyMQConfig.ORDER_EVENT_EXCHANGE,
                MyMQConfig.ORDER_RELEASE_OTHER_ROUTING_KEY,
                orderTo);
    } catch (Exception e) {
    
    }
    ```

- 消息抵达Broker, Broker要将消息写入磁盘(持久化)才算成功。此时Broker尚未持久化完成,宕机。

  - publisher也必须加入确认回调机制,确认成功的消息,修改数据库消息状态。

  - ```yaml
     spring:
       rabbitmq:
         host: 192.168.1.110
         port: 5672
         virtual-host: /
    #    开启发送端消息抵达 broker 的确认回调
         publisher-confirm-type: correlated
    #    开启发送端消息从 broker 抵达 queue 的确认回调【抵达失败回调，成功不回调】
         publisher-returns: true
    #    只要抵达 queue，以异步发送优先回调
         template:
           mandatory: true
    ```

    ```java
    /**
     * 定制 rabbitTemplate
     */
    @PostConstruct
    public void initRabbitTemplate() {
        rabbitTemplate.setConfirmCallback(new RabbitTemplate.ConfirmCallback() {
            /**
             * 设置消息抵达 broker 的确认回调
             * 消息抵达broker，ack为true
             * @param correlationData 当前消息的唯一关联数据
             * @param ack 消息是否成功收到
             * @param cause 失败的原因
             */
            @Override
            public void confirm(CorrelationData correlationData, boolean ack, String cause) {
                // TODO ack为true才说明消息成功到broker
                if (!ack) {
    
                } else {
    
                }
                System.out.println(correlationData + "," + ack + "," + cause);
            }
        });
    
        rabbitTemplate.setReturnCallback(new RabbitTemplate.ReturnCallback() {
            /**
             * 设置消息从 broker 抵达 queue 的确认回调
             * 【抵达失败回调，成功不回调】
             * @param message 投递失败的消息
             * @param replyCode 失败回复的状态吗
             * @param replyText 失败回复的原因
             * @param exchange  发给哪个交换机
             * @param routingKey 发送用的路由键
             */
            @Override
            public void returnedMessage(Message message, int replyCode, String replyText, String exchange, String routingKey) {
                // TODO 没发送到queue会回调这个方法
                System.out.println(message + "," + replyCode + "," + replyText + "," + exchange + "," + routingKey);
            }
        });
    }
    ```

- 自动ACK的状态下，消费者收到消息,但没来得及消息然后宕机

  - 一定开启手动ACK,消费成功才移除,失败或者没来得及处理就 noAck并重新入队

  - ```yaml
    spring:
      rabbitmq:
    #    消费端手动ack
        listener:
          simple:
            acknowledge-mode: manual
    ```

    ```java
    @RabbitListener(queues = MyMQConfig.ORDER_RELEASE_ORDER_QUEUE)
    @Service
    public class OrderCloseListener {
    
        @Resource
        private OrderService orderService;
    
        @RabbitHandler
        public void handleCloseOrder(OrderCreateTo orderCreateTo, Message message, Channel channel) throws IOException {
            System.out.println("收到关闭订单的信息");
            try {
                orderService.closeOrder(orderCreateTo);
                channel.basicAck(message.getMessageProperties().getDeliveryTag(), false);
            } catch (Exception e) {
                channel.basicReject(message.getMessageProperties().getDeliveryTag(), true);
            }
        }
    }
    ```



##### 2.消息重复
- 消息消费成功,事务已经提交,ack时机器宕机。导致没有ack成功,Broker的消息重新由unack变为 ready,并发送给其他消费者

- 消息消费失败,由于重试机制,自动又将消息发送出去

  - 消费者的业务消费接口应该设计为**幂等性**的。比如扣库存有工作单的状态标志

  - 使用**防重表**( redis/mysq),发送消息每一个都有业务的唯一标识,记录已经处理过的,处理过就不用处理

  - rabbitMQ的每一个消息都有redelivered字段,可以获取**是否是被重新投递过来的**,而不是第一次投递过来的



##### 3.消息积压

- 消费者宕机积压

- 消费者消费能力不足积压

- 发送者发送流量太大

  - 上线更多的消费者,进行正常消费

  - 上线专门的队列消费服务,先将消息批量取出来,记录数据库,离线慢慢处理