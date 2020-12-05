cron在线生成器

####  CronTrigger Tutorial

[CronTrigger Tutorial](http://www.quartz-scheduler.org/documentation/quartz-2.3.0/tutorials/crontrigger.html)

#### SpringBoot定时任务

```java
/**
 * 定时任务
 *   @EnableScheduling 开启定时任务功能
 *   @Scheduled        声明一个定时任务
 *   自动配置类 TaskSchedulingAutoConfiguration
 * 异步任务
 *   @EnableAsync       开始异步任务
 *   @Async             声明一个异步任务
 *   自动配置类 TaskExecutionAutoConfiguration
 *
 * 使用异步任务实现定时任务不阻塞
 *
 */
@Slf4j
@Component
@EnableScheduling
@EnableAsync
public class HelloScheduled {

    /**
     * spring中cron表达式6位组成，不支持第7位的年
     * 1-7代表周一到周日，直接写MON—SUN肯定不会错
     */
    @Async
    @Scheduled(cron = "* * * * * ?")
    public void hello() {
        log.info("hello");
        try { TimeUnit.SECONDS.sleep(3); } catch (InterruptedException e) { e.printStackTrace(); }
    }
}
```

```yaml
# 配置springboot提供的线程池
spring:
  task:
    execution:
      pool:
        core-size: 5
        max-size: 50
        queue-capacity: 100000
```



![](https://gitee.com/enioy/img/raw/master/K8S/20201205083358.png) 





![](https://gitee.com/enioy/img/raw/master/K8S/20201205172648.png)



 

