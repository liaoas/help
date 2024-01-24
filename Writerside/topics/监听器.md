# Monitor

Spring Batch 提供了多种监听器 **Listener**，用于在任务处理过程中触发我们的逻辑代码。常用的监听器根据粒度从粗到细分别有：**Job**级别 `JobExecutionListener`、**Step** 级别 `StepExecutionListener`、**Chunk** 监听器 `ChunkListener`、**ItmeReader** 监听器 `ItemReadListener`、**ItemWriter** 监听器 `ItemWriterListener` 和 **ItemProcessor**监听器 `ItemProcessListener` 等。具体可以参考下表：

| 监听器 | 具体说明 |
| --- | --- |
| JobExecutionListener | 在 Job 开始之前（befpreJob）和之后（aflerJob）触发 |
| StepExecutionListener | 在 Step 开始之前（beforeStep）和之后（afterStep）触发 |
| ChunkListener | 在 Chunk 开始之前（beforeChunk)，之后（afterChunk）和错误后(afterChunkError）触发 |
| ItemReadListener | 在 Reader 开始之前（beforeRead），之后（afterRead）和错误之后（onReadError）触发 |
| ItemProcessListener | 在 Processor 开始之前（beforeProcess），之后(afterProcess）和错误之后（onProcessError）触发 |
| ItemWriterListener | 在 Writer 开始之前（beforeWrite），之后（afterWrite）和错误之后（onWriteError）触发 |

# 项目搭建

新建一个 Spring Boot 项目 版本为 `2.7.12-SNAPSHOT` ，项目名称为：`spring-boot-batch-listener` ，Maven 依赖项如下：

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

# 监听器

每种监听器可以通过一下两种方式使用：**接口实现** 和 **注解实现**

## 接口实现 Job 驱动器

新建 `MyJobExecutionListener` 类实现 `JobExecutionListener` 用以实现 Job 监听器：

```java
package com.liao.listener;

/**
 * <p>
 * 通过接口实现 Job 监听器
 * </p>
 *
 * @author LiAo
 * @since 2023-07-07
 */
@Component
public class MyJobExecutionListener implements JobExecutionListener {

    @Override
    public void beforeJob(JobExecution jobExecution) {
        System.out.println("Job 开始之前: " + jobExecution.getJobInstance().getJobName());
    }

    @Override
    public void afterJob(JobExecution jobExecution) {
        System.out.println("Job 开始之后: " + jobExecution.getJobInstance().getJobName());
    }
}
```

## 注解实现 Step 驱动器

通过注解的方式不需要实现接口，只需要在对应的方法上新增 `@BeforeStep`*、*`@AfterStep` 注解即可

```java
package com.liao.listener;

/**
 * <p>
 * 通过注解实现 Step 监听器
 * </p>
 *
 * @author LiAo
 * @since 2023-07-07
 */
@Component
public class MyStepExecutionListener {

    /**
     * 通过注解实现监听器，函数命名 必须依照 注解的注释说明的固定函数名称
     * 如  beforeStep
     */
    @BeforeStep
    public void beforeStep(StepExecution stepExecution) {
        System.out.println("Step 开始之前: " + stepExecution.getStepName());
    }

    @AfterStep
    public void afterStep(StepExecution stepExecution) {
        System.out.println("Step 开始之后 : " + stepExecution.getStepName());
    }
}
```

方法名称要符合注解的要求，如 `@BeforeStep` 注解方法名称就要为 `beforeStep`

```java
package org.springframework.batch.core.annotation;

/**
 * Marks a method to be called before a {@link Step} is executed, which comes
 * after a {@link StepExecution} is created and persisted, but before the first
 * item is read. <br>
 * <br>
 * Expected signature: void beforeStep({@link StepExecution} stepExecution)
 * 
 * @author Lucas Ward
 * @since 2.0
 * @see StepExecutionListener
 */
@Retention(RetentionPolicy.RUNTIME)
@Target({ElementType.METHOD})
public @interface BeforeStep {

}
```

源码注释所示：Expected signature: void **beforeStep** ({@link StepExecution} stepExecution)

继续创建 Chunk 监听器、ItemReader 监听器、ItemProcess 监听器、ItemWriter 监听器

## Chunk 监听器

```java
package com.liao.listener;

/**
 * <p>
 * 通过接口实现 Chunk 监听器
 * </p>
 *
 * @author LiAo
 * @since 2023-07-07
 */
@Component
public class MyChunkListener implements ChunkListener {

    @Override
    public void beforeChunk(ChunkContext context) {
        System.out.println("chunk 开始之前 : " + context.getStepContext().getStepName());
    }

    @Override
    public void afterChunk(ChunkContext context) {
        System.out.println("chunk 开始之后 : " + context.getStepContext().getStepName());
    }

    @Override
    public void afterChunkError(ChunkContext context) {
        System.out.println("chunk error 之后 : " + context.getStepContext().getStepName());
    }
}
```

## ItemReader 监听器

```java
package com.liao.listener;

/**
 * <p>
 * 通过接口实现 ItemReader 监听器
 * </p>
 *
 * @author LiAo
 * @since 2023-07-07
 */
@Component
public class MyItemReaderListener implements ItemReadListener<String> {
    @Override
    public void beforeRead() {
        System.out.println("ItemReader 开始之前 ");
    }

    @Override
    public void afterRead(String item) {
        System.out.println("ItemReader 开始之后 : " + item);
    }

    @Override
    public void onReadError(Exception ex) {
        System.out.println("ItemReader 错误之后 : " + ex.getMessage());
    }
}
```

## ItemProcess 监听器

```java
package com.liao.listener;

/**
 * <p>
 * 通过接口实现 Process 监听器
 * </p>
 *
 * @author LiAo
 * @since 2023-07-07
 */
@Component
public class MyItemProcessListener implements ItemProcessListener<String, String> {

    @Override
    public void beforeProcess(String item) {
        System.out.println("process 开始之前 : " + item);
    }

    @Override
    public void afterProcess(String item, String result) {
        System.out.println("process 开始之后 : " + item + " result: " + result);
    }

    @Override
    public void onProcessError(String item, Exception e) {
        System.out.println("process 错误之后 : " + item + " , error message: " + e.getMessage());
    }
}
```

## ItemWriter 监听器

```java
package com.liao.listener;

/**
 * <p>
 *  通过接口实现 ItemWriter 监听器
 * </p>
 *
 * @author LiAo
 * @since 2023-07-07
 */
@Component
public class MyItemWriterListener implements ItemWriteListener<String> {

    @Override
    public void beforeWrite(List<? extends String> items) {
        System.out.println("ItemWriter 开始之前 : " + items);
    }

    @Override
    public void afterWrite(List<? extends String> items) {
        System.out.println("ItemWriter 开始之后 : " + items);
    }

    @Override
    public void onWriteError(Exception exception, List<? extends String> items) {
        System.out.println("ItemWriter 错误之后 : " + items + " , error message: " + exception.getMessage());
    }
}
```

准备好上述监听器，然后新建监听器任务测试类 `ListenerTestJobDemo` 用于测试监听器，代码如下：

```java
package com.liao.job;

/**
 * <p>
 * Spring Batch 监听器测试
 * </p>
 *
 * @author LiAo
 * @since 2023-07-12
 */
@Configuration
public class ListenerTestJobDemo {

    @Autowired
    private JobBuilderFactory jobBuilderFactory;

    @Autowired
    private StepBuilderFactory stepBuilderFactory;

    @Autowired
    private MyJobExecutionListener myJobExecutionListener;

    @Autowired
    private MyStepExecutionListener myStepExecutionListener;

    @Autowired
    private MyChunkListener myChunkListener;

    @Autowired
    private MyItemReaderListener myItemReaderListener;

    @Autowired
    private MyItemProcessListener myItemProcessListener;

    @Autowired
    private MyItemWriterListener myItemWriterListener;

    @Bean
    public Job listenerTestJob(@Qualifier("listenerTestStep") Step step) {
        return jobBuilderFactory.get("listenerTestJob")
                .start(step)
                .listener(myJobExecutionListener)
                .build();
    }

    @Bean("listenerTestStep")
    public Step listenerTestStep(@Qualifier("reader") ItemReader<String> reader,
                                 @Qualifier("processor") ItemProcessor<String, String> processor) {
        return stepBuilderFactory.get("ListenerTestStep")
                .listener(myStepExecutionListener)
                .<String, String>chunk(2)
                .faultTolerant()
                .listener(myChunkListener)
                .reader(reader)
                .listener(myItemReaderListener)
                .processor(processor)
                .listener(myItemProcessListener)
                .writer(list -> list.forEach(System.out::println))
                .listener(myItemWriterListener)
                .build();
    }

    @Bean("reader")
    public ItemReader<String> reader() {
        List<String> data = Arrays.asList("java", "c++", "javascript", "python");
        return new SimpleReader(data);
    }

    @Bean("processor")
    public ItemProcessor<String, String> processor() {
        return item -> item + " language";
    }

}

class SimpleReader implements ItemReader<String> {
    private final Iterator<String> iterator;

    public SimpleReader(List<String> data) {
        this.iterator = data.iterator();
    }

    @Override
    public String read() throws ParseException, NonTransientResourceException {
        return iterator.hasNext() ? iterator.next() : null;
    }
}
```

运行代码，运行结果如下所示：

```java
2023-07-12 16:38:35.180  INFO 63328 --- [           main] o.s.b.c.l.support.SimpleJobLauncher      : Job: [SimpleJob: [name=listenerTestJob1689151114649]] launched with the following parameters: [{}]
Job 开始之前: listenerTestJob1689151114649
2023-07-12 16:38:35.206  INFO 63328 --- [           main] o.s.batch.core.job.SimpleStepHandler     : Executing step: [ListenerTestStep]
Step 开始之前: ListenerTestStep
chunk 开始之前 : ListenerTestStep
ItemReader 开始之前 
ItemReader 开始之后 : java
ItemReader 开始之前 
ItemReader 开始之后 : c++
process 开始之前 : java
process 开始之后 : java result: java language
process 开始之前 : c++
process 开始之后 : c++ result: c++ language
ItemWriter 开始之前 : [java language, c++ language]
java language
c++ language
ItemWriter 开始之后 : [java language, c++ language]
chunk 开始之后 : ListenerTestStep
chunk 开始之前 : ListenerTestStep
ItemReader 开始之前 
ItemReader 开始之后 : javascript
ItemReader 开始之前 
ItemReader 开始之后 : python
process 开始之前 : javascript
process 开始之后 : javascript result: javascript language
process 开始之前 : python
process 开始之后 : python result: python language
ItemWriter 开始之前 : [javascript language, python language]
javascript language
python language
ItemWriter 开始之后 : [javascript language, python language]
chunk 开始之后 : ListenerTestStep
chunk 开始之前 : ListenerTestStep
ItemReader 开始之前 
chunk 开始之后 : ListenerTestStep
Step 开始之后 : ListenerTestStep
2023-07-12 16:38:35.235  INFO 63328 --- [           main] o.s.batch.core.step.AbstractStep         : Step: [ListenerTestStep] executed in 28ms
Job 开始之后: listenerTestJob1689151114649
2023-07-12 16:38:35.243  INFO 63328 --- [           main] o.s.b.c.l.support.SimpleJobLauncher      : Job: [SimpleJob: [name=listenerTestJob1689151114649]] completed with the following parameters: [{}] and the following status: [COMPLETED] in 50ms
```

从上面的运行结果我们可以看出：

1. 证实了`chunk(2)` 表示每一批处理2个数据块；
2. Step里的执行顺序是 `read -> process -> writer`

# 聚合监听器

每种监听器可以通过对应的聚合类组合在一起，比如有多个 `JobExecutionListener` ，则可以使用 `CompositeJobExecutionListener` 聚合起来。上述介绍的监听器均有与之对应的 `CompositeXXXListener` 聚合类，这里只演示 `CompositeJobExecutionListener` ，其余的大致一样。

新建聚合监听器测试类 `CompositeJobExecutionListenerJobDemo` 用于测试：

```java
package com.liao.job;

/**
 * <p>
 * Spring Batch 测试监听器聚合
 * </p>
 *
 * @author LiAo
 * @since 2023-07-12
 */
@Configuration
public class CompositeJobExecutionListenerJobDemo {

    @Autowired
    private JobBuilderFactory jobBuilderFactory;

    @Autowired
    private StepBuilderFactory stepBuilderFactory;

    @Bean
    public Job compositeJobExecutionListenerJob(@Qualifier("compositeJobExecutionListener") CompositeJobExecutionListener compositeJobExecutionListener,
                                                @Qualifier("compositeJobExecutionListenerStep") Step step) {
        return jobBuilderFactory.get("compositeJobExecutionListenerJob")
                .start(step)
                .listener(compositeJobExecutionListener)
                .build();
    }

    @Bean("compositeJobExecutionListenerStep")
    public Step step() {
        return stepBuilderFactory.get("compositeJobExecutionListenerStep")
                .tasklet(((contribution, chunkContext) -> {
                    System.out.println("执行步骤");
                    return RepeatStatus.FINISHED;
                })).build();
    }

    @Bean("compositeJobExecutionListener")
    public CompositeJobExecutionListener compositeJobExecutionListener() {
        CompositeJobExecutionListener listener = new CompositeJobExecutionListener();

        JobExecutionListener jobExecutionListenerOne = new JobExecutionListener() {

            @Override
            public void beforeJob(JobExecution jobExecution) {
                System.out.println("任务监听器One，job 开始之前 : " + jobExecution.getJobInstance().getJobName());
            }

            @Override
            public void afterJob(JobExecution jobExecution) {
                System.out.println("任务监听器One，job 开始之后 : " + jobExecution.getJobInstance().getJobName());
            }
        };

        JobExecutionListener jobExecutionListenerTwo = new JobExecutionListener() {

            @Override
            public void beforeJob(JobExecution jobExecution) {
                System.out.println("任务监听器Two，job 开始之前 : " + jobExecution.getJobInstance().getJobName());
            }

            @Override
            public void afterJob(JobExecution jobExecution) {
                System.out.println("任务监听器Two，job 开始之后 : " + jobExecution.getJobInstance().getJobName());
            }
        };

        listener.setListeners(Arrays.asList(jobExecutionListenerOne, jobExecutionListenerTwo));

        return listener;
    }
}
```

启动项目控制台打印信息如下：

```java
2023-07-12 17:22:53.161  INFO 60808 --- [           main] o.s.b.c.l.support.SimpleJobLauncher      : Job: [SimpleJob: [name=compositeJobExecutionListenerJob]] launched with the following parameters: [{}]
任务监听器One，job 开始之前 : compositeJobExecutionListenerJob
任务监听器Two，job 开始之前 : compositeJobExecutionListenerJob
2023-07-12 17:22:53.203  INFO 60808 --- [           main] o.s.batch.core.job.SimpleStepHandler     : Executing step: [compositeJobExecutionListenerStep]
执行步骤
2023-07-12 17:22:53.230  INFO 60808 --- [           main] o.s.batch.core.step.AbstractStep         : Step: [compositeJobExecutionListenerStep] executed in 27ms
任务监听器Two，job 开始之后 : compositeJobExecutionListenerJob
任务监听器One，job 开始之后 : compositeJobExecutionListenerJob
2023-07-12 17:22:53.244  INFO 60808 --- [           main] o.s.b.c.l.support.SimpleJobLauncher      : Job: [SimpleJob: [name=compositeJobExecutionListenerJob]] completed with the following parameters: [{}] and the following status: [COMPLETED] in 58ms
2023-07-12 17:22:53.310  INFO 60808 --- [           main] o.s.b.c.l.support.SimpleJobLauncher      : Job: [SimpleJob: [name=listenerTestJob]] launched with the following parameters: [{}]
Job 开始之前: listenerTestJob
2023-07-12 17:22:53.320  INFO 60808 --- [           main] o.s.batch.core.job.SimpleStepHandler     : Step already complete or not restartable, so no action to execute: StepExecution: id=278, version=5, name=ListenerTestStep, status=COMPLETED, exitStatus=COMPLETED, readCount=4, filterCount=0, writeCount=4 readSkipCount=0, writeSkipCount=0, processSkipCount=0, commitCount=3, rollbackCount=0, exitDescription=
Job 开始之后: listenerTestJob
2023-07-12 17:22:53.325  INFO 60808 --- [           main] o.s.b.c.l.support.SimpleJobLauncher      : Job: [SimpleJob: [name=listenerTestJob]] completed with the following parameters: [{}] and the following status: [COMPLETED] in 10ms
```