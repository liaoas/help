# Exception

Spring Batch 处理任务过程中，如果遇到了异常，默认机制是马上停止任务执行，抛出相应异常。如果任务还有未执行的步骤也不会执行。若要改变这个规则，可以配置 **异常重试** 和 **异常跳过** 机制。

- **异常跳过**

  遇到异常的时候不希望结束任务，而是跳过这个异常，继续执行

- **异常重试**

  遇到异常时经过指定次数的重试，如果还是失败的话，才会停止任务。


# 项目搭建

新建一个 Spring Boot 项目 版本为 `2.7.12-SNAPSHOT` ，项目名称为：`spring-boot-batch-exception` ，Maven 依赖项如下：

```xml
		<dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter</artifactId>
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
spring:
  datasource:
    driver-class-name: com.mysql.cj.jdbc.Driver
    url: jdbc:mysql://127.0.0.1:3306/spring_batch_test
    username: root
    password: liao

```

至此，基本框架搭建好了，下面开始配置一个简单的任务。

# 默认异常处理

新建任务测试类 `DefaultExceptionJobDemo` ，用于测试默认情况下 Spring Batch 下默认处理异常机制

```java
package com.liao.job;

/**
 * <p>
 * Spring Batch 默认异常处理测试
 * </p>
 *
 * @author LiAo
 * @since 2023-07-13
 */
@Configuration
public class DefaultExceptionJobDemo {

    @Resource
    private JobBuilderFactory jobBuilderFactory;

    @Resource
    private StepBuilderFactory stepBuilderFactory;

    @Bean
    public Job defaultExceptionJob() {
        return jobBuilderFactory.get("defaultExceptionJob")
                .start(
                        stepBuilderFactory.get("defaultExceptionStep")
                                .tasklet((stepContribution, chunkContext) -> {
                                    ExecutionContext execution = chunkContext.getStepContext().getStepExecution().getExecutionContext();
                                    if (execution.containsKey("success")) {
                                        System.out.println("任务执行成功");
                                        return RepeatStatus.FINISHED;
                                    } else {
                                        String errorMessage = "处理任务过程发生异常";
                                        System.out.println(errorMessage);
                                        execution.put("success", true);
                                        throw new RuntimeException(errorMessage);
                                    }
                                }).build()
                ).build();
    }
}
```

上述代码，在 Step 的 tasklet() 函数中获取执行上下文，并判断 上下文中是否包含 success，如果包含则执行成功；如果不包含，则抛出异常，并在抛出异常前在上下文中添加 “success” 这个 key。

启动项目，控制台打印如下：

```java
2023-07-13 16:58:16.941  INFO 19100 --- [           main] o.s.batch.core.job.SimpleStepHandler     : Executing step: [defaultExceptionStep]
处理任务过程发生异常
2023-07-13 16:58:16.956 ERROR 19100 --- [           main] o.s.batch.core.step.AbstractStep         : Encountered an error executing step defaultExceptionStep in job defaultExceptionJob

java.lang.RuntimeException: 处理任务过程发生异常
	at com.liao.job.DefaultExceptionJobDemo.lambda$defaultExceptionJob$0(DefaultExceptionJobDemo.java:45) ~[classes/:na]
	at org.springframework.batch.core.step.tasklet.TaskletStep$ChunkTransactionCallback.doInTransaction(TaskletStep.java:407) ~[spring-batch-core-4.3.8.jar:4.3.8]
	at org.springframework.batch.core.step.tasklet.TaskletStep$ChunkTransactionCallback.doInTransaction(TaskletStep.java:331) ~[spring-batch-core-4.3.8.jar:4.3.8]
	at org.springframework.transaction.support.TransactionTemplate.execute(TransactionTemplate.java:140) ~[spring-tx-5.3.27.jar:5.3.27]
	at org.springframework.batch.core.step.tasklet.TaskletStep$2.doInChunkContext(TaskletStep.java:273) ~[spring-batch-core-4.3.8.jar:4.3.8]
	at org.springframework.batch.core.scope.context.StepContextRepeatCallback.doInIteration(StepContextRepeatCallback.java:82) ~[spring-batch-core-4.3.8.jar:4.3.8]
```

根据上述控制台打印来看，Spring Batch 执行任务时如果发生了异常会马上停止任务的执行

再次启动项目打印如下：

```java
2023-07-13 17:05:44.309  INFO 2128 --- [           main] o.s.batch.core.job.SimpleStepHandler     : Executing step: [defaultExceptionStep]
任务执行成功
2023-07-13 17:05:44.323  INFO 2128 --- [           main] o.s.batch.core.step.AbstractStep         : Step: [defaultExceptionStep] executed in 14ms
2023-07-13 17:05:44.331  INFO 2128 --- [           main] o.s.b.c.l.support.SimpleJobLauncher      : Job: [SimpleJob: [name=defaultExceptionJob]] completed with the following parameters: [{}] and the following status: [COMPLETED] in 34ms
```

因为在上次任务抛出异常前，我们在执行上下文中添加 `success` key（配合MySQL持久化，不会因项目启动而丢失）。

# 异常重试

Spring Batch 允许配置任务遇到指定异常时的重试次数。在此之前。需要先定义一个异常 `MyJobExecutionException`

```java
package com.liao.exception;

/**
 * <p>
 * 自定义异常
 * </p>
 *
 * @author LiAo
 * @since 2023-07-14
 */
public class MyJobExecutionException extends Exception {

    private static final long serialVersionUID = 7168487913507656106L;

    public MyJobExecutionException(String message) {
        super(message);
    }
}
```

然后新建异常重试测试任务类 `RetryExceptionJobDemo`

```java
package com.liao.job;

/**
 * <p>
 * Spring Batch 异常重试测试
 * </p>
 *
 * @author LiAo
 * @since 2023-07-14
 */
@Configuration
public class RetryExceptionJobDemo {

    @Autowired
    private JobBuilderFactory jobBuilderFactory;

    @Autowired
    private StepBuilderFactory stepBuilderFactory;

    @Bean
    public Job job(@Qualifier("RetryExceptionStep") Step step) {
        return jobBuilderFactory.get("RetryExceptionJob")
                .start(step)
                .build();
    }

    @Bean("RetryExceptionStep")
    public Step step(@Qualifier("RetryExceptionItemReader") ListItemReader<String> listItemReader,
                     @Qualifier("RetryExceptionProcessor") ItemProcessor<String, String> processor) {
        return stepBuilderFactory.get("RetryExceptionStep")
                .<String, String>chunk(2)
                .reader(listItemReader)
                .processor(processor)
                .writer(items -> items.forEach(System.out::println))
                .faultTolerant() // 配置错误容忍
                .retry(MyJobExecutionException.class) // 配置异常类型
                // 重试次数，三次过后还会异常，则结束任务
                // 异常的次数为reader，processor和writer中的总数，这里仅在processor里演示异常重试
                .retryLimit(3)
                .build();
    }

    @Bean("RetryExceptionItemReader")
    public ListItemReader<String> listItemReader() {
        List<String> list = new ArrayList<>();

        IntStream.range(0, 5).forEach(i -> list.add(String.valueOf(i)));

        return new ListItemReader<>(list);
    }

    @Bean("RetryExceptionProcessor")
    public ItemProcessor<String, String> myProcessor() {
        return new ItemProcessor<String, String>() {
            private int count;

            @Override
            public String process(String item) throws Exception {
                System.out.println("当前处理的数据：" + item);
                if (count >= 2) {
                    return item;
                } else {
                    count++;
                    throw new MyJobExecutionException("任务处理出错");
                }
            }

        };
    }
}
```

在 `step()` 函数中，`faultTolerant()` 函数表示开启容错功能，`retry(MyJobExecutionException.class)` 表示遇到 `MyJobExecutionException` 异常时进行重试

`myProcessor()` 函数的代码逻辑是：在前两次的时候抛出 `MyJobExecutionException("任务处理出错")` 的异常，第三次的时候正常返回 `item` ，理论上，上述任务在报错重试两次之后正常运行。

启动项目，日志打印如下：

```java
2023-07-14 11:05:45.923  INFO 13420 --- [           main] o.s.batch.core.job.SimpleStepHandler     : Executing step: [SkipExceptionStep1689303945313]
当前处理的数据：0
当前处理的数据：0
当前处理的数据：0
当前处理的数据：1
0
1
当前处理的数据：2
当前处理的数据：3
2
3
当前处理的数据：4
4
2023-07-14 11:05:45.950  INFO 13420 --- [           main] o.s.batch.core.step.AbstractStep         : Step: [SkipExceptionStep1689303945313] executed in 27ms
```

若是把 `retryLimit` 改为 `retryLimit(2)` ，并修改任务名称为 `RetryExceptionJob01` ，启动项目再次测试运行结果，如下：

```java
2023-07-26 17:16:28.526  INFO 24760 --- [           main] o.s.batch.core.job.SimpleStepHandler     : Executing step: [RetryExceptionStep1690362987642]
当前处理的数据：0
当前处理的数据：0
2023-07-26 17:16:28.549 ERROR 24760 --- [           main] o.s.batch.core.step.AbstractStep         : Encountered an error executing step RetryExceptionStep1690362987642 in job RetryExceptionJob01

org.springframework.retry.RetryException: Non-skippable exception in recoverer while processing; nested exception is com.liao.exception.MyJobExecutionException: 任务处理出错
	at org.springframework.batch.core.step.item.FaultTolerantChunkProcessor$2.recover(FaultTolerantChunkProcessor.java:299) ~[spring-batch-core-4.3.8.jar:4.3.8]
```

上述结果是异常重试超过了重试次数，抛出了异常

# 异常跳过

可以在 Step 中配置异常跳过，即遇到指定类型的异常时忽略跳过他，首先新建自定义 `SkipListener` 监听器 `MySkipListener` 用于打印异常跳过时的异常信息：

```java
package com.liao.listener;

/**
 * <p>
 * Skip 类型监听器
 * </p>
 *
 * @author LiAo
 * @since 2023-07-26
 */
@Component
public class MySkipListener implements SkipListener<String, String> {

    @Override
    public void onSkipInRead(Throwable t) {
        System.out.println("在读取数据的时候遇到异常并跳过，异常：" + t.getMessage());
    }

    @Override
    public void onSkipInWrite(String item, Throwable t) {
        System.out.println("在输出数据的时候遇到异常并跳过，待输出数据：" + item + "，异常：" + t.getMessage());
    }

    @Override
    public void onSkipInProcess(String item, Throwable t) {
        System.out.println("在处理数据的时候遇到异常并跳过，待输出数据：" + item + "，异常：" + t.getMessage());
    }
}
```

上述新建了一个自定义监听器，用于打印遇到异常时的 值 和 和异常信息

新建测试类 `SkipExceptionJobDemo` 用于测试 异常跳过

```java
package com.liao.job;

/**
 * <p>
 * Spring Boot 异常跳过测试
 * </p>
 *
 * @author LiAo
 * @since 2023-07-26
 */
@Configuration
public class SkipExceptionJobDemo {

    @Autowired
    private JobBuilderFactory jobBuilderFactory;

    @Autowired
    private StepBuilderFactory stepBuilderFactory;

    @Autowired
    private MySkipListener mySkipListener;

    @Bean
    public Job skipExceptionJob(@Qualifier("SkipExceptionStep") Step step) {
        return jobBuilderFactory.get("skipExceptionJob" + System.currentTimeMillis())
                .start(step)
                .build();
    }

    @Bean("SkipExceptionStep")
    public Step step(@Qualifier("SkipExceptionProcessor") ItemProcessor<String, String> processor,
                     @Qualifier("SkipExceptionReader") ListItemReader<String> reader) {
        return stepBuilderFactory.get("skipExceptionStep")
                .<String, String>chunk(2)
                .reader(reader)
                .processor(processor)
                .writer(items -> items.forEach(System.out::println))
                .faultTolerant()//配置错误容忍
                .skip(MyJobExecutionException.class)// 配置跳过的异常类型
                // 异常的次数为reader，processor和writer中的总数，这里仅在processor里演示异常跳过
                .skipLimit(1)// 最多跳过1次，1次过后还是异常的话，则任务会结束，
                // 自定义监听器，用于打印异常信息
                .listener(mySkipListener)
                .build();
    }

    @Bean("SkipExceptionReader")
    public ListItemReader<String> listItemReader() {
        List<String> strings = new ArrayList<>();

        IntStream.range(0, 5).forEach(i -> strings.add(String.valueOf(i)));

        return new ListItemReader<>(strings);
    }

    @Bean("SkipExceptionProcessor")
    public ItemProcessor<String, String> myProcessor() {
        return item -> {
            System.out.println("当前处理的数据" + item);
            if ("2".equals(item)) {
                throw new MyJobExecutionException("任务处理出错");
            } else {
                return item;
            }
        };
    }
}
```

在 `step()` 函数中，`faultTolerant()` 表示开启容错功能，`skip(MyJobExecutionException.class)` 表示遇到 `MyJobExecutionException` 异常时跳过，`skipLimit(1)` 表示只跳过一次。

`myProcessor()` 的逻辑时，当处理的 item 值为 “2” 的时候，抛出 `MyJobExecutionException("任务处理出错")` 的异常。

`listener(mySkipListener)` 用于注入一个自定义的监听器

启动项目，日志打印如下：

```java
2023-07-27 14:57:59.417  INFO 8184 --- [           main] o.s.batch.core.job.SimpleStepHandler     : Executing step: [skipExceptionStep]
当前处理的数据0
当前处理的数据1
0
1
当前处理的数据2
当前处理的数据3
3
在处理数据的时候遇到异常并跳过，待输出数据：2，异常：任务处理出错
当前处理的数据4
4
2023-07-27 14:57:59.436  INFO 8184 --- [           main] o.s.batch.core.step.AbstractStep         : Step: [skipExceptionStep] executed in 19ms
```

# 事务问题

一次 **Step** 分为 **Reader**、**Processor**、**Writer** 三个阶段，这些阶段统称为 **Itme**。默认情况下如果错误不是发生在 **Reader** 阶段，那么就没必要重新去读取一次数据。但是某些场景下 **Reader** 部分也需要重新执行，比如 **Reader** 是从一个 **JMS** 队列中消费消息，当发生回滚的时候也会在队列上重放，因此也要将 **Reader** 纳入到回滚的事务中，根据这个场景可以使用 `readerIsTransactionalQueue()` 来配置数据重读。

```java
private Step step() {
    return stepBuilderFactory.get("step")
            .<String, String>chunk(2)
            .reader(listItemReader())
            .writer(list -> list.forEach(System.out::println))
            .readerIsTransactionalQueue() // 消息队列数据重读
            .build();
}
```

也可以在 `Step` 中手动配置事务属性，事物的属性包括 **隔离等级**（isolation）、**传播方式**（propagation）以及 **过期时间**（timeout）等：

```java
private Step step() {
    DefaultTransactionAttribute attribute = new DefaultTransactionAttribute();
    attribute.setPropagationBehavior(Propagation.REQUIRED.value());
    attribute.setIsolationLevel(Isolation.DEFAULT.value());
    attribute.setTimeout(30);

    return stepBuilderFactory.get("step")
            .<String, String>chunk(2)
            .reader(listItemReader())
            .writer(list -> list.forEach(System.out::println))
            .transactionAttribute(attribute)
            .build();
}
```

# 重启机制

默认情况下，任务执行完毕的状态为 COMPLETED ，再次启动项目，该任务的 Step 不会再次执行，我们可以配置 `allowStartIfComplete(true)` 来实现每次项目启动都执行这个 **Step:**

```java
private Step step() {
    return stepBuilderFactory.get("step")
            .<String, String>chunk(2)
            .reader(listItemReader())
            .writer(list -> list.forEach(System.out::println))
            .allowStartIfComplete(true)
            .build();
}
```

某些 **Step** 可能用于处理一些先决的任务，所以当 **Job** 再次重启时这 `Step` 就没必要再执行，可以通过设置 `startLimit()` 来限定某个 **Step** 重启的次数。当设置为 1 时候表示仅仅运行一次，而出现重启时将不再执行：

```java
private Step step() {
    return stepBuilderFactory.get("step")
            .<String, String>chunk(2)
            .reader(listItemReader())
            .writer(list -> list.forEach(System.out::println))
            .startLimit(1)
            .build();
}
```