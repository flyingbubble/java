### springboot的3中实现方式

#### 一、基于注解(@Scheduled)；

```
@Component
@Configuration      //1.主要用于标记配置类，兼备Component的效果。
@EnableScheduling   // 2.开启定时任务
public class ScheduleTask {
    //3.添加定时任务
    @Scheduled(cron = "0/5 * * * * ?")
    //或直接指定时间间隔，例如：5秒
    //@Scheduled(fixedRate=5000)
    private void configureTasks() {
        System.err.println("执行静态定时任务时间: " + LocalDateTime.now());
    }
}
```

Cron表达式：

秒（0~59） 例如0/5表示每5秒；分（0~59）；时（0~23）；日（0~31）的某天，需计算；月（0~11）；周几（ 可填1-7 或 SUN/MON/TUE/WED/THU/FRI/SAT）；

@Scheduled：除了支持灵活的参数表达式cron之外，还支持简单的延时操作，如 fixedDelay ，fixedRate 填写相应的毫秒数即可

优点:@Scheduled 注解很方便

缺点:是当我们调整了执行周期的时候，需要重启应用才能生效。为了达到实时生效的效果，可以使用接口来完成定时任务

\------------------

#### 二、基于接口（SchedulingConfigurer） 

但是实际使用中我们往往想从数据库中读取指定时间来动态执行定时任务，这时候基于接口的定时任务就派上用场:

需要操作数据库（导入mybatis依赖）：

```mysql
CREATE TABLE `cron`  (
  `cron_id` varchar(30) NOT NULL PRIMARY KEY,
  `cron` varchar(30) NOT NULL  
);
INSERT INTO `cron` VALUES ('1', '0/5 * * * * ?');
```



```java
@Component
@Configuration
@EnableScheduling 
public class DynamicScheduleTask implements SchedulingConfigurer {

  @Mapper
  public interface CronMapper {
    @Select("select cron from cron limit 1")
    public String getCron();
  }

  @Autowired
  CronMapper cronMapper;
  @Override
  public void configureTasks(ScheduledTaskRegistrar taskRegistrar) {

    taskRegistrar.addTriggerTask(
        //1.添加任务内容(Runnable)
        () -> System.out.println("执行动态定时任务: " + LocalDateTime.now().toLocalTime()),
        //2.设置执行周期(Trigger)
        triggerContext -> {
          //2.1 从数据库获取执行周期
          String cron = cronMapper.getCron();
          //2.2 合法性校验.
          if (StringUtils.isEmpty(cron)) {
            // Omitted Code ..
          }
          //2.3 返回执行周期(Date)
          return new CronTrigger(cron).nextExecutionTime(trigger
                                                         Context);
        });
  }
}
```

#### 三、基于注解设定多线程定时任务

```java
@Component
@EnableScheduling
@EnableAsync    // 2.开启多线程
public class MultithreadScheduleTask {

    @Async
    @Scheduled(fixedDelay = 1000) 
    public void first() throws InterruptedException {
      System.out.println("第一个定时任务开始 : " + LocalDateTime.now().toLocalTime() + "\r\n线程 : " + Thread.currentThread().getName());
      System.out.println();
      Thread.sleep(1000 * 10);
    }

    @Async
    @Scheduled(fixedDelay = 2000)
    public void second() {
      System.out.println("第二个定时任务开始 : " + LocalDateTime.now().toLocalTime() + "\r\n线程 : " + Thread.currentThread().getName());
      System.out.println();
    }
}
```

