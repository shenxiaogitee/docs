### Feign远程调用丢失请求头问题

问题描述：feign在远程调用的时候，会创建一个request来请求远程方法，远程方法在获取请求头中cookie的时候，发现拿到的是null

断点调试：发现feign创建的request中headers为空

![feign06](https://gitee.com/enioy/img/raw/master/docs/20201122215054.png)

查看源码发现，在创建request前，会遍历RequestInterceptor并调用apply方法，默认容器中是没有RequestInterceptor的

```java
final class SynchronousMethodHandler implements MethodHandler {
  ...
  Object executeAndDecode(RequestTemplate template, Options options) throws Throwable {
    Request request = targetRequest(template);
    ...
  }
  ...
  Request targetRequest(RequestTemplate template) {
    for (RequestInterceptor interceptor : requestInterceptors) {
      interceptor.apply(template);
    }
    return target.apply(template);
  }
  ...
}
```

解决办法：给容器中注入RequestInterceptor，从而在创建request前，把老请求的请求头的cookie放入到新request中

```java
@Configuration
public class FeignConfig {
    @Bean
    public RequestInterceptor requestInterceptor() {
        return new RequestInterceptor() {
            @Override
            public void apply(RequestTemplate template) {
                ServletRequestAttributes servletRequestAttributes = (ServletRequestAttributes) RequestContextHolder.getRequestAttributes();
                if (servletRequestAttributes != null) {
                    HttpServletRequest request = servletRequestAttributes.getRequest();
                    template.header("Cookie", request.getHeader("Cookie"));
                }
            }
        };
    }
}
```

再次调试，请求头同步成功

![feign08](https://gitee.com/enioy/img/raw/master/docs/20201122215139.png)

总结：

![](https://gitee.com/enioy/img/raw/master/docs/20201122214328.png)

![](https://gitee.com/enioy/img/raw/master/docs/20201122214901.png)

### Feign异步调用丢失请求头问题

问题描述：在上面的基础上，当使用线程池来异步编排远程调用时，发现请求头又没了

![](https://gitee.com/enioy/img/raw/master/docs/20201122214952.png)

原因分析：上面获取老的请求，是通过RequestContextHolder.getRequestAttributes();查看源码，其实使用的是ThreadLocal，由于使用了线程池，导致异步调用时线程发生了改变，从而拿不到老的请求

![feign09](https://gitee.com/enioy/img/raw/master/docs/20201122215152.png)

解决办法：在异步调用开始前，把当前线程的requestAttributes()同步到异步线程中

```java
RequestAttributes requestAttributes = RequestContextHolder.getRequestAttributes();

RequestContextHolder.setRequestAttributes(requestAttributes);
```

 ![feign10](https://gitee.com/enioy/img/raw/master/docs/20201122215203.png)

总结：

![](https://gitee.com/enioy/img/raw/master/docs/20201122215005.png)

