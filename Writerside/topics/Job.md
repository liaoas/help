# Job

# 项目搭建

新建一个 Spring Boot 项目 版本为 `2.7.12-SNAPSHOT` ，项目名称为：`spring-boot-batch-launcher` ，Maven 依赖项如下：

```xml
			<dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter</artifactId>
        </dependency>

        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>

        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-batch</artifactId>
        </dependency>

        <dependency>
            <groupId>mysql</groupId>
            <artifactId>mysql-connector-java</artifactId>
            <version>8.0.32</version>
        </dependency>

        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-test</artifactId>
            <scope>test</scope>
        </dependency>
```

## 启动类

`Spring Boot` 启动类添加 `@EnableBatchProcessing`

```java
@EnableBatchProcessing
@SpringBootApplication
public class AppApplication {

    public static void main(String[] args) {
        SpringApplication.run(AppApplication.class, args);
    }

}
```

## 数据库准备

准备一个 MySQL 数据库，用于持久化 Spring Batch 的任务，新建一个数据库，数据库命名为 `spring_batch_test` 并导入 Spring Batch 所用到的数据表，org.springframework.batch.core目录下的schema-mysql.sql文件：

## 项目配置

配置项目的数据库连接信息如下：

```yaml
server:
  port: 80
spring:
  datasource:
    driver-class-name: com.mysql.cj.jdbc.Driver
    url: jdbc:mysql://127.0.0.1:3306/spring_batch_test
    username: root
    password: liao
```

至此，基本框架搭建好了，下面开始配置一个简单的任务。

此次示例需要通过 Controller 里通过 JobLauncher 或 JobOperator 调度任务，所以我们需要引入 `spring-boot-starter-web` 依赖，由于 `spring-boot-starter-web` 依赖中包含了 `spring-boot-starter` 依赖，所以可以将 `spring-boot-starter` 依赖剔除

```xml
				<dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>
```

依赖准备就绪，准备测试任务类 `MyJob` ，用于后续测试

```java
package com.liao.job;

/**
 * <p>
 * Spring Batch 任务测试
 * </p>
 *
 * @author LiAo
 * @since 2023-07-27
 */
@Configuration
public class MyJob {

    @Autowired
    private JobBuilderFactory jobBuilderFactory;

    @Autowired
    private StepBuilderFactory stepBuilderFactory;

    @Bean
    public Job job(@Qualifier("MyJobStep") Step step) {
        return jobBuilderFactory.get("job")
                .start(step)
                .build();
    }

    @Bean("MyJobStep")
    public Step step() {
        return stepBuilderFactory.get("MyJobStep")
                .tasklet((stepContribution, chunkContext) -> {
                    StepExecution stepExecution = chunkContext.getStepContext().getStepExecution();

                    Map<String, JobParameter> parameters = stepExecution.getJobParameters().getParameters();

                    System.out.println(parameters.get("message").getValue());

                    return RepeatStatus.FINISHED;
                })
                .listener(this)
                .build();
    }
}
```

在 `step()` 函数中，通过执行上下文获取了 **key** 为 **message** 的参数值

# ****JobLauncher****

新建 Controller 类 `JobController` 用于测试

```java
package com.liao.controller;

/**
 * <p>
 *
 * </p>
 *
 * @author LiAo
 * @since 2023-07-27
 */
@RestController
@RequestMapping("job")
public class JobController {

    // 注入自定义任务测试类 MyJob类中的 Job
    @Autowired
    private Job job;

    @Autowired
    private JobLauncher jobLauncher;

    @GetMapping("launcher/{message}")
    public String launcher(@PathVariable String message) throws Exception {
        JobParameters parameters = new JobParametersBuilder()
                .addString("message", message)
                .toJobParameters();
        jobLauncher.run(job, parameters);

        return message;
    }
}
```

上面代码中，我们注入了`JobLauncher`和上面配置的`Job`，然后通过`JobLauncher`的`run(Job job, JobParameters jobParameters)`方法运行指定的任务 Job，并且传递了参数。

要关闭Spring Batch启动项目自动运行任务的机制，需要在项目配置文件application.yml中添加如下配置：

```yaml
spring:
	batch:
    job:
      enabled: false
```

启动项目，访问如下地址：

```bash
curl http://localhost/job/launcher/hello
hello
```

控制台日志打印如下：

```json
2023-08-03 15:26:22.243  INFO 16808 --- [p-nio-80-exec-1] o.s.b.c.l.support.SimpleJobLauncher      : Job: [SimpleJob: [name=job]] launched with the following parameters: [{message=hello}]
2023-08-03 15:26:22.275  INFO 16808 --- [p-nio-80-exec-1] o.s.batch.core.job.SimpleStepHandler     : Executing step: [MyJobStep]
hello
2023-08-03 15:26:22.292  INFO 16808 --- [p-nio-80-exec-1] o.s.batch.core.step.AbstractStep         : Step: [MyJobStep] executed in 16ms
2023-08-03 15:26:22.301  INFO 16808 --- [p-nio-80-exec-1] o.s.b.c.l.support.SimpleJobLauncher      : Job: [SimpleJob: [name=job]] completed with the following parameters: [{message=hello}] and the following status: [COMPLETED] in 38ms
```

此外，需要注意的是：同样的参数，同样的任务再次运行的时候将抛出`JobInstanceAlreadyCompleteException`异常，比如在浏览器中再次访问 http://localhost/job/launcher/hello，项目控制台日志打印如下：

```java
org.springframework.batch.core.repository.JobInstanceAlreadyCompleteException: A job instance already exists and is complete for parameters={message=hello}.  If you want to run this job again, change the parameters.
	at org.springframework.batch.core.repository.support.SimpleJobRepository.createJobExecution(SimpleJobRepository.java:139) ~[spring-batch-core-4.3.8.jar:4.3.8]
	at sun.reflect.NativeMethodAccessorImpl.invoke0(Native Method) ~[na:1.8.0_131]
	at sun.reflect.NativeMethodAccessorImpl.invoke(NativeMethodAccessorImpl.java:62) ~[na:1.8.0_131]
	at sun.reflect.DelegatingMethodAccessorImpl.invoke(DelegatingMethodAccessorImpl.java:43) ~[na:1.8.0_131]
	at java.lang.reflect.Method.invoke(Method.java:498) ~[na:1.8.0_131]
	at org.springframework.aop.support.AopUtils.invokeJoinpointUsingReflection(AopUtils.java:344) ~[spring-aop-5.3.27.jar:5.3.27]
	at org.springframework.aop.framework.ReflectiveMethodInvocation.invokeJoinpoint(ReflectiveMethodInvocation.java:198) ~[spring-aop-5.3.27.jar:5.3.27]
	at org.springframework.aop.framework.ReflectiveMethodInvocation.proceed(ReflectiveMethodInvocation.java:163) ~[spring-aop-5.3.27.jar:5.3.27]
	at org.springframework.transaction.interceptor.TransactionInterceptor$1.proceedWithInvocation(TransactionInterceptor.java:123) ~[spring-tx-5.3.27.jar:5.3.27]
```

所以我们在任务调度的时候，应避免参数重复。

# ****JobOperator****

在`JobController`里添加一个新的端点：

```java
package com.liao.controller;

/**
 * <p>
 *
 * </p>
 *
 * @author LiAo
 * @since 2023-07-27
 */
@RestController
@RequestMapping("job")
public class JobController {

    @Autowired
    private JobOperator jobOperator;

    @GetMapping("operator/{message}")
    public String operator(@PathVariable String message) throws Exception {
        // 传递任务名称，参数使用 kv方式
        jobOperator.start("job", "message=" + message);
        return message;
    }
}
```

上面代码中，我们注入了`JobOperator`，`JobOperator`的*`s*tart(String jobName, String parameters)`方法传入的是任务的名称（任务在Spring IOC容器中的名称）,并且参数使用key-value的方式传递。

要通过任务名称获取到相应的Bean，还需要添加一个额外的配置。新建`JobConfigure` 配置类

```java
package com.liao.configure;

/**
 * <p>
 * Spring Batch 任务配置类
 * </p>
 *
 * @author LiAo
 * @since 2023-08-03
 */
@Configuration
public class JobConfigure {

    /**
     * 注册 JobRegistryBeanPostProcessor 用于将任务名称和实际任务关联起来
     *
     * @param jobRegistry        任务
     * @param applicationContext applicationContext
     * @return JobRegistryBeanPostProcessor
     */
    @Bean
    public JobRegistryBeanPostProcessor postProcessor(JobRegistry jobRegistry, ApplicationContext applicationContext) {
        JobRegistryBeanPostProcessor po = new JobRegistryBeanPostProcessor();

        po.setJobRegistry(jobRegistry);
        po.setBeanFactory(applicationContext.getAutowireCapableBeanFactory());

        return po;
    }
}
```

## 注意！！！

如果没有上述配置，在任务调度的时候将报 `org.springframework.batch.core.launch.NoSuchJobException: No job configuration with the name [job] was registered` 。

启动后访问测试接口

```bash
curl http://localhost/job/operator/world
world
```

项目控制台日志打印如下：

```java
2023-08-03 17:14:59.536  INFO 17312 --- [p-nio-80-exec-4] o.s.b.c.l.support.SimpleJobOperator      : Attempting to launch job with name=job and parameters=message=hello1
2023-08-03 17:14:59.575  INFO 17312 --- [p-nio-80-exec-4] o.s.b.c.l.support.SimpleJobLauncher      : Job: [SimpleJob: [name=job]] launched with the following parameters: [{message=hello1}]
2023-08-03 17:14:59.607  INFO 17312 --- [p-nio-80-exec-4] o.s.batch.core.job.SimpleStepHandler     : Executing step: [MyJobStep]
hello1
2023-08-03 17:14:59.626  INFO 17312 --- [p-nio-80-exec-4] o.s.batch.core.step.AbstractStep         : Step: [MyJobStep] executed in 19ms
2023-08-03 17:14:59.635  INFO 17312 --- [p-nio-80-exec-4] o.s.b.c.l.support.SimpleJobLauncher      : Job: [SimpleJob: [name=job]] completed with the following parameters: [{message=hello1}] and the following status: [COMPLETED] in 39ms
```