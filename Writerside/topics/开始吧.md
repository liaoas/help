# 开始吧

# 项目搭建

新建一个 Spring Boot 项目 版本为 `2.7.12-SNAPSHOT` ，项目名称为：`spring-boot-batch` ，Maven 依赖项如下所示：

```xml
	<dependencies>
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
            <groupId>org.springframework.batch</groupId>
            <artifactId>spring-batch-test</artifactId>
            <scope>test</scope>
        </dependency>
    </dependencies>
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

# 编写任务

新建 job 包，并新建一个任务类如下：

```java
package com.liao.job;

/**
 * <p>
 * 测试执行单个步骤任务
 * </p>
 *
 * @author LiAo
 * @since 2023-05-11
 */
@Component
public class FirstJobDemo {

    @Autowired
    private JobBuilderFactory jobBuilderFactory;

    @Autowired
    private StepBuilderFactory stepBuilderFactory;

    @Bean
    public Job firstJob01() {
        return jobBuilderFactory.get("firstJob01").start(step01()).build();
    }

    public Step step01() {
        return stepBuilderFactory.get("step01").tasklet((contribution, chunkContext) -> {
            System.out.println("Spring Batch 执行第一个步骤...");
            // 处理完成
            return RepeatStatus.FINISHED;
        }).build();
    }
}
```

上述代码使用，`JobBuilderFactory` 任务工厂和 `StepBuilderFactory`  步骤工厂，分别用于创建，任务和步骤，`JobBuilderFactory.get` 创建一个具体的任务名称，`start` 方法用于指定开始步骤，步骤通过 `StepBuilderFactory` 构建。步骤 Step 由若干个小任务 Testklet 组成。通过 tasklet 方法创建任务，tasklet 方法接收一个 Tasklet 类型的参数，Tasklet 是一个函数式接口，源码如下：

```java
public interface Tasklet {
		@Nullable
		RepeatStatus execute(StepContribution contribution, ChunkContext chunkContext) throws Exception;
}
```

所以可以使用 Lambda 表达式实现 Tasklet 匿名函数的创建：

```java
(contribution, chunkContext) -> {
		System.out.println("Spring Batch 执行第一个步骤...");
		// 处理完成
		return RepeatStatus.FINISHED;
}
```

该匿名函数返回值为 RepeatStatus ，表示任务的执行状态，这里使用 `RepeatStatus.FINISHED` 表示该小任务执行完成，正常结束。

配置的任务必须注册到 Spring IOC 容器中，并且任务名称和步骤名称必须唯一。比如上面的例子，任务名称为 `firstJob01` 步骤名称为 `step01` 如果别的任务也叫这个名称的话，则会执行失败。启动项目，控制台打印如下

```java
Spring Batch 执行第一个步骤...
```

可以看到，定义的任务执行成功，数据库中也有响应的记录

重启项目后，控制台并不会再次打印出任务执行日志，因为Job名称和 Step名称组成唯一，执行完的不可重复的任务，不会再次执行。

# 多步骤任务

一般情况下一个复杂的任务包含多个任务步骤，下列代码所示多步骤任务创建：

```java
package com.liao.job;

/**
 * <p>
 * Spring Batch 多步骤任务
 * </p>
 *
 * @author LiAo
 * @since 2023-05-11
 */
@Component
public class MultiStepJobDemo {

    @Autowired
    private JobBuilderFactory jobBuilderFactory;

    @Autowired
    private StepBuilderFactory stepBuilderFactory;

    @Bean
    private Job multiStepJob01() {
        return jobBuilderFactory.get("multiStepJob01")
                .start(step01())
                .next(step02())
                .next(step03())
                .build();
    }

    public Step step01() {
        return stepBuilderFactory.get("step01").tasklet((contribution, chunkContext) -> {
            System.out.println("MultiStepJobDemo Spring Batch 多步骤任务测试 步骤1 step01...");
            return RepeatStatus.FINISHED;
        }).build();
    }

    public Step step02() {
        return stepBuilderFactory.get("step02").tasklet((contribution, chunkContext) -> {
            System.out.println("MultiStepJobDemo Spring Batch 多步骤任务测试 步骤1 step02...");
            return RepeatStatus.FINISHED;
        }).build();
    }

    public Step step03() {
        return stepBuilderFactory.get("step03").tasklet((contribution, chunkContext) -> {
            System.out.println("MultiStepJobDemo Spring Batch 多步骤任务测试 步骤1 step03...");
            return RepeatStatus.FINISHED;
        }).build();
    }
}
```

上面代码中，我们通过*`step1()`*、*`step2()`*和*`step3()`*三个方法创建了三个步骤。Job里要使用这些步骤，只需要通过*`JobBuilderFactory`*的*`start`*方法指定第一个步骤，然后通过*`next`*方法不断地指定下一个步骤。

启动项目打印结果如下：

```java
MultiStepJobDemo Spring Batch 多步骤任务测试 步骤1 step01...
2023-05-11 16:31:05.877  INFO 2948 --- [           main] o.s.batch.core.step.AbstractStep         : Step: [step01] executed in 18ms
2023-05-11 16:31:05.889  INFO 2948 --- [           main] o.s.batch.core.job.SimpleStepHandler     : Executing step: [step02]
MultiStepJobDemo Spring Batch 多步骤任务测试 步骤1 step02...
2023-05-11 16:31:05.901  INFO 2948 --- [           main] o.s.batch.core.step.AbstractStep         : Step: [step02] executed in 12ms
2023-05-11 16:31:05.917  INFO 2948 --- [           main] o.s.batch.core.job.SimpleStepHandler     : Executing step: [step03]
MultiStepJobDemo Spring Batch 多步骤任务测试 步骤1 step03...
2023-05-11 16:31:05.928  INFO 2948 --- [           main] o.s.batch.core.step.AbstractStep         : Step: [step03] executed in 11ms
2023-05-11 16:31:05.936  INFO 2948 --- [           main] o.s.b.c.l.support.SimpleJobLauncher      : Job: [SimpleJob: [name=multiStepJob01]] completed with the following parameters: [{}] and the following status: [COMPLETED] in 88ms
```

三个步骤执行成功

# 多步骤任务状态判断

多步骤任务在执行过程中也可以通过上一个步骤的执行状态来判定是否执行下一个步骤，修改上面 `multiStepJob01` 函数的代码如下所示：

```java
		@Bean
    public Job multiStepJob01() {
        return jobBuilderFactory.get("multiStepJob01")
                .start(step01())
                .on(ExitStatus.COMPLETED.getExitCode()).to(step02())
                .from(step02())
                .on(ExitStatus.COMPLETED.getExitCode()).to(step03())
                .from(step03()).end()
                .build();
    }

    public Step step01() {
        return stepBuilderFactory.get("step01").tasklet((contribution, chunkContext) -> {
            System.out.println("MultiStepJobDemo Spring Batch 多步骤任务测试 步骤1 step01...");
            return RepeatStatus.FINISHED;
        }).build();
    }

    public Step step02() {
        return stepBuilderFactory.get("step02").tasklet((contribution, chunkContext) -> {
            System.out.println("MultiStepJobDemo Spring Batch 多步骤任务测试 步骤2 step02...");
            return RepeatStatus.FINISHED;
        }).build();
    }

    public Step step03() {
        return stepBuilderFactory.get("step03").tasklet((contribution, chunkContext) -> {
            System.out.println("MultiStepJobDemo Spring Batch 多步骤任务测试 步骤3 step03...");
            return RepeatStatus.FINISHED;
        }).build();
    }
```

`multiStepJob01` 方法的含义是：首先执行 step01，当 step01 执行状态为完成时，接着执行 step02 , 当 step02 执行状态为完成时，接着执行step3。`ExitStatus.COMPLETED`常量表示任务顺利执行完毕，正常退出。

## 注意

上述代码在 `Spring Batch 4.2.5.RELEASE` 及其以上时，不会执行 `**step03**` 步骤，因为从 `Spring Batch 4.2.5.RELEASE` 开始，`org.springframework.batch.core.job.builder.FlowBuilder<Q>.createState(Object)` 函数做了如下变动：

```java
states.put(input, new StepState(prefix + step.getName(), step));                // Spring Batch 4.2.4.RELEASE 及其以前
states.put(input, new StepState(prefix + "step" + (stepCounter++), step));      // Spring Batch 4.2.5.RELEASE 及其以后
```

## Spring Batch 4.2.5.RELEASE 代码变动

上述代码的变动，导致 `from()` 函数中构建的内容只能是 `Spring IoC` 容器中的，否则就会失效，导致 `from` 往下的流程不执行，所以上述 [**多步骤状态判断](https://www.notion.so/057885c326f749258e43b3927c22037b?pvs=21)** 代码需要做如下变动

```java
package com.liao.job;

/**
 * <p>
 * Spring Batch 多步骤任务
 * </p>
 *
 * @author LiAo
 * @since 2023-05-11
 */
@Configuration
public class MultiStepJobDemo {

    @Autowired
    private JobBuilderFactory jobBuilderFactory;

    @Autowired
    private StepBuilderFactory stepBuilderFactory;

    @Bean
    public Job multiStepJob02(@Qualifier("multiStepJobStep01") Step stop01,
                              @Qualifier("multiStepJobStep02") Step step02,
                              @Qualifier("multiStepJobStep03") Step step03) {
        return jobBuilderFactory.get("multiStepJob02")
                .start(stop01)
                .on(ExitStatus.COMPLETED.getExitCode()).to(step02)
                .from(step02())
                .on(ExitStatus.COMPLETED.getExitCode()).to(step03)
                .from(step03).end()
                .build();
    }

    @Bean("multiStepJobStep01")
    public Step step01() {
        return stepBuilderFactory.get("step01").tasklet((contribution, chunkContext) -> {
            System.out.println("MultiStepJobDemo Spring Batch 多步骤任务测试 步骤1 step01...");
            return RepeatStatus.FINISHED;
        }).build();
    }

    @Bean("multiStepJobStep02")
    public Step step02() {
        return stepBuilderFactory.get("step02").tasklet((contribution, chunkContext) -> {
            System.out.println("MultiStepJobDemo Spring Batch 多步骤任务测试 步骤2 step02...");
            return RepeatStatus.FINISHED;
        }).build();
    }

    @Bean("multiStepJobStep03")
    public Step step03() {
        return stepBuilderFactory.get("step03").tasklet((contribution, chunkContext) -> {
            System.out.println("MultiStepJobDemo Spring Batch 多步骤任务测试 步骤3 step03...");
            return RepeatStatus.FINISHED;
        }).build();
    }
}
```

上述代码将 Step 步骤通过 `@Bean` 注解注册到 `Spring IoC` 容器中，然后再进行 Job 任务的构建

`Spring Batch 4.2.4.RELEASE` 及其之前版本支持 **函数名称** 和 **容器对象** 两种方式的调用，从 `Spring Batch 4.2.5.RELEASE` 版本开始，只支持容器对象这一种方式。 `Spring Boot` 从 `2.3.7.RELEASE` 版本开始使用 `Spring Batch 4.2.5.RELEASE`

# Flow 任务

Flow 的作用是将多个 step 步骤组装起来，然后再组装到任务Job中。下列代码是 Flow 任务示例：

```java
package com.liao.job;

/**
 * <p>
 * Spring Batch Flow 流程任务测试
 * </p>
 *
 * @author LiAo
 * @since 2023-05-12
 */
@Configuration
public class FlowJobDemo {

    @Autowired
    private JobBuilderFactory jobBuilderFactory;

    @Autowired
    private StepBuilderFactory stepBuilderFactory;

    @Bean
    public Job flowJob01(@Qualifier("flowJobFlow01") Flow flow01,
                         @Qualifier("flowJobStep03") Step step03) {
        return jobBuilderFactory.get("flowJob")
                .start(flow01)
                .next(step03).end().build();
    }

    @Bean("flowJobStep01")
    public Step step01() {
        return stepBuilderFactory.get("step01").tasklet((contribution, chunkContext) -> {
            System.out.println("FlowJobDemo epJobDemo Spring Batch Flow任务测试 步骤1 step01...");
            return RepeatStatus.FINISHED;
        }).build();
    }

    @Bean("flowJobStep02")
    public Step step02() {
        return stepBuilderFactory.get("step02").tasklet((contribution, chunkContext) -> {
            System.out.println("FlowJobDemo Spring Batch Flow任务测试 步骤2 step02...");
            return RepeatStatus.FINISHED;
        }).build();
    }

    @Bean("flowJobStep03")
    public Step step03() {
        return stepBuilderFactory.get("step03").tasklet((contribution, chunkContext) -> {
            System.out.println("FlowJobDemo Spring Batch Flow任务测试 步骤3 step03...");
            return RepeatStatus.FINISHED;
        }).build();
    }

    @Bean("flowJobFlow01")
    public Flow flow01(@Qualifier("flowJobStep01") Step step01,
                       @Qualifier("flowJobStep02") Step step02) {
        return new FlowBuilder<Flow>("flow01")
                .start(step01)
                .next(step02)
                .build();
    }
}
```

上述代码通过 `FlowBuilder` 将步骤 step01 和 step02 组合在一起，然后再将其赋给任务 flowJob ，使用 Flow 和 Step 构建 Job 的区别是：Job 任务流程中包含 Flow 任务流程时，在调用 `build()` 方法之前需要调用 `end()` 方法

运行代码执行结果如下所示：

```java
FlowJobDemo Spring Batch Flow任务测试 步骤1 step01...
2023-05-12 15:03:49.575  INFO 23988 --- [           main] o.s.batch.core.step.AbstractStep         : Step: [step01] executed in 19ms
2023-05-12 15:03:49.591  INFO 23988 --- [           main] o.s.batch.core.job.SimpleStepHandler     : Executing step: [step02]
FlowJobDemo Spring Batch Flow任务测试 步骤2 step02...
2023-05-12 15:03:49.602  INFO 23988 --- [           main] o.s.batch.core.step.AbstractStep         : Step: [step02] executed in 10ms
2023-05-12 15:03:49.621  INFO 23988 --- [           main] o.s.batch.core.job.SimpleStepHandler     : Executing step: [step03]
FlowJobDemo Spring Batch Flow任务测试 步骤3 step03...
2023-05-12 15:03:49.633  INFO 23988 --- [           main] o.s.batch.core.step.AbstractStep         : Step: [step03] executed in 12ms
```

# 并行任务

Spring Batch 任务中除了串行执行任务（一个执行结束执行下一个），也可以并行执行（两个一起执行），并执行在特定的业务下可以提高执行的效率，将任务并行化有如下两个步骤：

1. 将步骤 Step 转换为 Flow
2. 在任务中指定需要并行的 Flow

并行任务如下所示：

```java
package com.liao.job;

/**
 * <p>
 * Spring Batch 并行任务测试
 * </p>
 *
 * @author LiAo
 * @since 2023-05-12
 */
@Configuration
public class SplitJobDemo {

    @Autowired
    private JobBuilderFactory jobBuilderFactory;
    @Autowired
    private StepBuilderFactory stepBuilderFactory;

    @Bean
    public Job splitJob01(@Qualifier("splitJobFlow01") Flow flow01,
                          @Qualifier("splitJobFlow02") Flow flow02) {
        return jobBuilderFactory.get("splitJob")
                .start(flow01)
                .split(new SimpleAsyncTaskExecutor()).add(flow02)
                .end().build();
    }

    @Bean("splitJobStep01")
    public Step step01() {
        return stepBuilderFactory.get("step01").tasklet((contribution, chunkContext) -> {
            System.out.println("SplitJobDemo Spring Batch Split任务测试 步骤1 step01...");
            return RepeatStatus.FINISHED;
        }).build();
    }

    @Bean("splitJobStep02")
    public Step step02() {
        return stepBuilderFactory.get("step02").tasklet((contribution, chunkContext) -> {
            System.out.println("SplitJobDemo Spring Batch Split任务测试 步骤2 step02...");
            return RepeatStatus.FINISHED;
        }).build();
    }

    @Bean("splitJobStep03")
    public Step step03() {
        return stepBuilderFactory.get("step03").tasklet((contribution, chunkContext) -> {
            System.out.println("SplitJobDemo Spring Batch Split任务测试 步骤3 step03...");
            return RepeatStatus.FINISHED;
        }).build();
    }

    @Bean("splitJobFlow01")
    public Flow flow01(@Qualifier("splitJobStep01") Step step01,
                       @Qualifier("splitJobStep02") Step step02) {
        return new FlowBuilder<Flow>("flow01")
                .start(step01)
                .next(step02)
                .build();
    }

    @Bean("splitJobFlow02")
    public Flow flow02(@Qualifier("splitJobStep03") Step step03) {
        return new FlowBuilder<Flow>("flow02")
                .start(step03)
                .build();
    }
}
```

上述代码创建了两个 Flow：flow01 包含 step01 和 step02，flow02 包含 step03，然后通过 `JobBuilderFactory.split()` 方法，指定一个异步执行器，将 flow01 和 flow02 并行执行。

启动项目日志打印如下：

```java
2023-05-12 15:46:02.747  INFO 26068 --- [cTaskExecutor-1] o.s.batch.core.job.SimpleStepHandler     : Executing step: [step03]
2023-05-12 15:46:02.750  INFO 26068 --- [cTaskExecutor-2] o.s.batch.core.job.SimpleStepHandler     : Executing step: [step01]
SplitJobDemo Spring Batch Split任务测试 步骤3 step03...
SplitJobDemo Spring Batch Split任务测试 步骤1 step01...
2023-05-12 15:46:02.768  INFO 26068 --- [cTaskExecutor-1] o.s.batch.core.step.AbstractStep         : Step: [step03] executed in 21ms
2023-05-12 15:46:02.768  INFO 26068 --- [cTaskExecutor-2] o.s.batch.core.step.AbstractStep         : Step: [step01] executed in 18ms
2023-05-12 15:46:02.788  INFO 26068 --- [cTaskExecutor-2] o.s.batch.core.job.SimpleStepHandler     : Executing step: [step02]
SplitJobDemo Spring Batch Split任务测试 步骤2 step02...
2023-05-12 15:46:02.801  INFO 26068 --- [cTaskExecutor-2] o.s.batch.core.step.AbstractStep         : Step: [step02] executed in 12ms
```

可以看到 step03 和 step01 先后执行，然后才执行 step02 说明并行任务执行成功，并行任务的顺序不能百分百确定，因为线程的调度具有不确定性

# 任务决策器

决策器用于决定任务在不同情况下运行不同流程，若今天是周末执行 step01 步骤，否则就执行 step02 步骤。

## 决策器创建

决策器使用之前需要创建一个决策器类，并实现 `org.springframework.batch.core.job.flow` 包下的 `JobExecutionDecider` 接口的 `decide` 方法：

```java
package com.liao.decider;

/**
 * <p>
 * Spring Batch 任务决策器
 * </p>
 *
 * @author LiAo
 * @since 2023-05-12
 */
@Component
public class MyDecider implements JobExecutionDecider {

    /**
     * 自定义任务决策器
     *
     * @param jobExecution  a job execution
     * @param stepExecution the latest step execution (maybe {@code null})
     * @return the exit status code
     */
    @Override
    public FlowExecutionStatus decide(JobExecution jobExecution, StepExecution stepExecution) {
        LocalDate now = LocalDate.now();
        DayOfWeek dayOfWeek = now.getDayOfWeek();
        if (dayOfWeek == DayOfWeek.SATURDAY || dayOfWeek == DayOfWeek.SUNDAY) {
            return new FlowExecutionStatus("weekend");
        } else {
            return new FlowExecutionStatus("workingDay");
        }
    }
}
```

MyDecider 类实现了 `JobExecutionDecider`  接口的 `decide` 方法，改方法返回一个 `FlowExecutionStatus` ，上述代码逻辑为：若今天为周末则返回  `FlowExecutionStatus("weekend")` 否则返回 `FlowExecutionStatus("workingDay")`

## 决策器使用

```java
package com.liao.job;

/**
 * <p>
 * 测试 Spring Batch 任务决策器
 * </p>
 *
 * @author LiAo
 * @since 2023-05-12
 */
@Configuration
public class DeciderJobDemo {

    @Autowired
    private JobBuilderFactory jobBuilderFactory;

    @Autowired
    private StepBuilderFactory stepBuilderFactory;

    @Autowired
    private MyDecider myDecider;

    @Bean
    public Job deciderJob01(@Qualifier("step01") Step step01,
                            @Qualifier("step02") Step step02,
                            @Qualifier("step03") Step step03,
                            @Qualifier("step04") Step step04) {
        return jobBuilderFactory.get("deciderJob01")
                .start(step01)
                .next(myDecider)
                .from(myDecider)
                .on("weekend").to(step02)
                .from(myDecider)
                .on("workingDay").to(step03)
                .from(step03)
                .on("*").to(step04)
                .end().build();
    }

    @Bean("step01")
    public Step step01() {
        return stepBuilderFactory.get("step01").tasklet((contribution, chunkContext) -> {
            System.out.println("DeciderJobDemo Spring Batch Decider Split任务测试 步骤1 step01...");
            return RepeatStatus.FINISHED;
        }).build();
    }

    @Bean(name = "step02")
    public Step step02() {
        return stepBuilderFactory.get("step02").tasklet((contribution, chunkContext) -> {
            System.out.println("DeciderJobDemo Spring Batch Decider Split任务测试 weekend 步骤2 step02...");
            return RepeatStatus.FINISHED;
        }).build();
    }

    @Bean(name = "step03")
    public Step step03() {
        return stepBuilderFactory.get("liao03").tasklet((contribution, chunkContext) -> {
            System.out.println("DeciderJobDemo Spring Batch Decider Split任务测试 workingDay 步骤3 step03...");
            return RepeatStatus.FINISHED;
        }).build();
    }

    @Bean(name = "step04")
    public Step step04() {
        return stepBuilderFactory.get("step04").tasklet((contribution, chunkContext) -> {
            System.out.println("DeciderJobDemo Spring Batch Decider Split任务测试 * 步骤4 step04...");
            return RepeatStatus.FINISHED;
        }).build();
    }
}
```

上述代码的含义是：名为 `deciderJob01` 的任务首先执行 `step01` 然后指定自定义的决策器，若决策器返回 `weekend` 则执行 `step02`，若是返回 `workingDay` 则执行 `step03`，若是执行了 `step03` 则无论结果如何都执行 `step04`

启动项目，任务执行结果如下所示：

```java
2023-05-17 10:37:42.281  INFO 10640 --- [           main] o.s.batch.core.job.SimpleStepHandler     : Executing step: [step01]
MultiStepJobDemo Spring Batch Decider Split任务测试 步骤1 step01...
2023-05-17 10:37:42.306  INFO 10640 --- [           main] o.s.batch.core.step.AbstractStep         : Step: [step01] executed in 23ms
2023-05-17 10:37:42.330  INFO 10640 --- [           main] o.s.batch.core.job.SimpleStepHandler     : Executing step: [liao03]
MultiStepJobDemo Spring Batch Decider Split任务测试 workingDay 步骤3 step03...
2023-05-17 10:37:42.345  INFO 10640 --- [           main] o.s.batch.core.step.AbstractStep         : Step: [liao03] executed in 15ms
2023-05-17 10:37:42.365  INFO 10640 --- [           main] o.s.batch.core.job.SimpleStepHandler     : Executing step: [step04]
MultiStepJobDemo Spring Batch Decider Split任务测试 * 步骤4 step04...
```

因为上述代码的执行日期为 `2023-05-17` 是周三 所以执行了 step01、step03、step04 这三个步骤。

# 嵌套任务

任务 Job 除了可以由 Step 个 Flow 构成之外，还可以将 Job 转为特殊的 Step，然后再构建进另一个任务 Job ，这就是嵌套任务：

```java
package com.liao.job;

/**
 * <p>
 * Spring Batch 任务嵌套测试
 * </p>
 *
 * @author LiAo
 * @since 2023-05-17
 */
@Configuration
public class NestedJobDemo {

    @Autowired
    private JobBuilderFactory jobBuilderFactory;

    @Autowired
    private StepBuilderFactory stepBuilderFactory;

    @Autowired
    private JobLauncher jobLauncher;

    @Autowired
    private JobRepository jobRepository;

    @Autowired
    private PlatformTransactionManager platformTransactionManager;

    @Bean
    public Job parentJob(@Qualifier("nestedChildJobOneStep") Step childJobOneStep,
                         @Qualifier("nestedChildJobTowStep") Step childJobTowStep) {
        return jobBuilderFactory.get("parentJob")
                .start(childJobOneStep)
                .next(childJobTowStep)
                .build();
    }

    // 将任务转为特殊的 Step
    @Bean("nestedChildJobOneStep")
    public Step childJobOneStep(@Qualifier("nestedJobChildJobOne") Job childJobOne) {
        return new JobStepBuilder(new StepBuilder("nestedChildJobOneStep"))
                .job(childJobOne)
                .launcher(jobLauncher)
                .repository(jobRepository)
                .transactionManager(platformTransactionManager)
                .build();
    }

    // 将任务转为特殊的 Step
    @Bean("nestedChildJobTowStep")
    public Step childJobTowStep(@Qualifier("nestedJobChildJobTow") Job childJobTwo) {
        return new JobStepBuilder(new StepBuilder("nestedChildJobTwoStep"))
                .job(childJobTwo)
                .launcher(jobLauncher)
                .repository(jobRepository)
                .transactionManager(platformTransactionManager)
                .build();
    }

    // 嵌套子任务1
    @Bean("nestedJobChildJobOne")
    public Job childJobOne() {
        return jobBuilderFactory.get("nestedJobChildJobOne")
                .start(
                        stepBuilderFactory.get("nestedJobChildJobOneStep")
                                .tasklet((stepContribution, chunkContext) -> {
                                    System.out.println("NestedJobDemo Spring Batch Nested 嵌套子任务执行 childJobOne 任务1 ...");
                                    return RepeatStatus.FINISHED;
                                }).build()
                ).build();
    }

    // 嵌套子任务2
    @Bean("nestedJobChildJobTow")
    public Job childJobTwo() {
        return jobBuilderFactory.get("nestedJobChildJobTwo")
                .start(
                        stepBuilderFactory.get("nestedJobChildJobTwoStep")
                                .tasklet((stepContribution, chunkContext) -> {
                                    System.out.println("NestedJobDemo Spring Batch Nested 嵌套子任务执行 childJobTwo 任务2 ...");
                                    return RepeatStatus.FINISHED;
                                }).build()
                ).build();
    }
}
```

上述代码首先通过 `childJobOne()` 和 `childJobTwo()` 函数构建了两个普通任务，分别为 `nestedJobChildJobOne` 和 `nestedJobChildJobTow`，然后在 `childJobOneStep()` 函数中，通过 `org.springframework.batch.core.step.builder.JobStepBuider` 构建了一个名为 `nestedChildJobTwoStep` 的特殊 Step，`JobStepBuilder` 望名知意，它是一个 `Job` 类型的 `Setp` 构建器，可以将 Job 构建为 Step，在构建的时候，需要传入 任务执行器 `org.springframework.batch.core.launch.JobLauncher` 和 任务仓库 `org.springframework.batch.core.repository.JobRepository` 以及事务管理器 `org.springframework.transaction.PlatformTransactionManager`

将转换好的特殊步骤构建进最终的任务 `parentJob` 中即可

配置好后运行代码如下所示：

```java
NestedJobDemo Spring Batch Nested 嵌套子任务执行 childJobOne 任务1 ...
2023-05-17 11:19:09.088  INFO 18168 --- [           main] o.s.batch.core.step.AbstractStep         : Step: [nestedJobChildJobOneStep] executed in 15ms
2023-05-17 11:19:09.098  INFO 18168 --- [           main] o.s.b.c.l.support.SimpleJobLauncher      : Job: [SimpleJob: [name=nestedJobChildJobOne]] completed with the following parameters: [{}] and the following status: [COMPLETED] in 34ms
2023-05-17 11:19:09.129  INFO 18168 --- [           main] o.s.batch.core.step.AbstractStep         : Step: [nestedChildJobOneStep] executed in 87ms
2023-05-17 11:19:09.144  INFO 18168 --- [           main] o.s.batch.core.job.SimpleStepHandler     : Executing step: [nestedChildJobTwoStep]
2023-05-17 11:19:09.158  INFO 18168 --- [           main] o.s.b.c.l.support.SimpleJobLauncher      : Job: [SimpleJob: [name=nestedJobChildJobTwo]] launched with the following parameters: [{}]
2023-05-17 11:19:09.172  INFO 18168 --- [           main] o.s.batch.core.job.SimpleStepHandler     : Executing step: [nestedJobChildJobTwoStep]
NestedJobDemo Spring Batch Nested 嵌套子任务执行 childJobTwo 任务2 ...
```..