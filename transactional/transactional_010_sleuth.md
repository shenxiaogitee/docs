### 链路追踪



sleuth

![](https://gitee.com/enioy/img/raw/master/K8S/20201211110455.png) 

```
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-redis</artifactId>
    <exclusions>
        <!-- lettuce-core 有内存泄露问题，切换为 jedis -->
        <exclusion>
            <groupId>io.lettuce</groupId>
            <artifactId>lettuce-core</artifactId>
        </exclusion>
    </exclusions>
</dependency>
<dependency>
    <groupId>redis.clients</groupId>
    <artifactId>jedis</artifactId>
</dependency>
```

使用redis要排除lettuce，引入sleuth后，不排除lettuce无法启动



引入zipkin

![](https://gitee.com/enioy/img/raw/master/K8S/20201211113735.png) 



![](https://gitee.com/enioy/img/raw/master/K8S/20201211115317.png) 



![](https://gitee.com/enioy/img/raw/master/K8S/20201211115459.png) 

![](https://gitee.com/enioy/img/raw/master/K8S/20201211115538.png) 

![](https://gitee.com/enioy/img/raw/master/K8S/20201211115639.png) 

