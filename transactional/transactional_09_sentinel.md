#### ### sentinel



#### 一：sentinel使用

1.引依赖

```pom
<dependency>
    <groupId>com.alibaba.cloud</groupId>
    <artifactId>spring-cloud-starter-alibaba-sentinel</artifactId>
</dependency>
```

2.写配置

```yaml
spring:
    sentinel:
      transport:
        port: 8719
        dashboard: localhost:8333
```

3.启动dashboard，java -jar sentinel-dashboard-1.8.0.jar --server.port=8333

4.控制台配置限流

![](https://gitee.com/enioy/img/raw/master/K8S/20201210170311.png)



5.测试正常页面

![](https://gitee.com/enioy/img/raw/master/K8S/20201210164707.png) 

 

快速刷新触发限流

![](https://gitee.com/enioy/img/raw/master/K8S/20201210170451.png) 

6.自定义响应

```java
@Configuration
public class SentinelBlockExceptionHandler implements BlockExceptionHandler {

    @Override
    public void handle(HttpServletRequest request, HttpServletResponse response, BlockException e) throws Exception {
        R r = R.error();
        if (e instanceof FlowException) {
            r.put("msg", "触发限流规则");
        }
        if (e instanceof DegradeException) {
            r.put("msg", "触发降级规则");
        }
        if (e instanceof ParamFlowException) {
            r.put("msg", "触发热点规则");
        }
        if (e instanceof SystemBlockException) {
            r.put("msg", "触发系统规则");
        }
        if (e instanceof AuthorityException) {
            r.put("msg", "触发授权规则");
        }
        response.setCharacterEncoding("utf-8");
        response.setContentType("application/json;charset=utf-8");
        response.getWriter().write(JSON.toJSONString(r));

    }
}
```



7.测试，快速刷新页面

![](https://gitee.com/enioy/img/raw/master/K8S/20201210171554.png) 





#### 二：使用sentinel保护feign的远程调用,熔断降级



##### 1:调用方的熔断保护

商品满足秒杀条件时商品页面

![](https://gitee.com/enioy/img/raw/master/K8S/20201210164707.png) 



product微服务商品详情页面，会远程调用seckill微服务查询当前商品的秒杀信息，如果秒杀服务宕机了，则会显示错误页面，商品页面都没法显示



![](https://gitee.com/enioy/img/raw/master/K8S/20201210162407.png) 



远程调用代码：

```java
@FeignClient("gulimall-seckill")
@Service
public interface SeckillFeignService {

    @GetMapping("/sku/seckill/{skuId}")
    R getSkuSeckillInfo(@PathVariable("skuId") Long skuId);
}
```



![image-20201210162302213](https://gitee.com/enioy/img/raw/master/K8S/20201210162426.png) 



修改product微服务

```yaml
feign:
  sentinel:
    enabled: true
```

```java
@FeignClient(value = "gulimall-seckill", fallback = SeckillFeignServiceFallback.class)
@Service
public interface SeckillFeignService {

    @GetMapping("/sku/seckill/{skuId}")
    R getSkuSeckillInfo(@PathVariable("skuId") Long skuId);
}
```

```java
@Slf4j
@Component
public class SeckillFeignServiceFallback implements SeckillFeignService {
    @Override
    public R getSkuSeckillInfo(Long skuId) {
        log.info("熔断方法调用");
        return R.error();
    }
}
```

这样即使seckill微服务宕机了，远程调用会走熔断方法，页面能正常显示，虽然没有秒杀信息，但是能正常购买



![](https://gitee.com/enioy/img/raw/master/K8S/20201210163249.png) 





##### 2:调用方手动指定远程服务的降级策略

还是商品页面，有秒杀信息

![](https://gitee.com/enioy/img/raw/master/K8S/20201210164707.png)  



控制台product微服务下配置远程调用的降级策略



![](https://gitee.com/enioy/img/raw/master/K8S/20201210164524.png) 



![](https://gitee.com/enioy/img/raw/master/K8S/20201210164303.png) 



秒杀信息模拟延时，这样虽然seckill微服务没有宕机，但是由于延时触发了降级规则，就会触发降级

```java
/**
 * sku秒杀信息
 */
@ResponseBody
@GetMapping("/sku/seckill/{skuId}")
public R getSkuSeckillInfo(@PathVariable("skuId") Long skuId) {
    try { TimeUnit.MILLISECONDS.sleep(300); } catch (InterruptedException e) { e.printStackTrace(); }
    SeckillSkuRedisTo to = seckillService.getSkuSeckillInfo(skuId);
    return R.ok().setData(to);
}
```



![](https://gitee.com/enioy/img/raw/master/K8S/20201210163249.png) 





#### 三：网关限流

1.引依赖

```yaml
<dependency>
    <groupId>com.alibaba.cloud</groupId>
    <artifactId>spring-cloud-alibaba-sentinel-gateway</artifactId>
    <version>2.2.2.RELEASE</version>
</dependency>
```



2.控制台效果

![](https://gitee.com/enioy/img/raw/master/K8S/20201211090901.png) 

3.测试

![](https://gitee.com/enioy/img/raw/master/K8S/20201211092058.png) 

默认限流返回

![](https://gitee.com/enioy/img/raw/master/K8S/20201211092130.png) 

您可以在 `GatewayCallbackManager` 注册回调进行定制：

- `setBlockHandler`：注册函数用于实现自定义的逻辑处理被限流的请求，对应接口为 `BlockRequestHandler`。默认实现为 `DefaultBlockRequestHandler`，当被限流时会返回类似于下面的错误信息：`Blocked by Sentinel: FlowException`。



自定义返回

```java
@Configuration
public class SentinelGatewayConfig {

    public SentinelGatewayConfig() {
        GatewayCallbackManager.setBlockHandler(new BlockRequestHandler() {
            // 网管限流回调
            @Override
            public Mono<ServerResponse> handleRequest(ServerWebExchange serverWebExchange, Throwable throwable) {
                R r = R.error();
                String s = JSON.toJSONString(r);
                Mono<ServerResponse> body = ServerResponse.ok().body(Mono.just(s), String.class);
                return body;
            }
        });
    }
}
```



![](https://gitee.com/enioy/img/raw/master/K8S/20201211095127.png)



