---
title: Spring Task Execution and Scheduling
date: 2017-08-02 10:29:58
tags: Spring
---
# Spring Task Execution and Scheduling

Spring框架分别提供了使用TaskExecutor和TaskScheduler接口的异步执行和任务调度的抽象。Spring还具有支持线程池或在应用程序服务器环境中对CommonJ进行委派的那些接口的实现。最终在公共接口背后使用这些实现可以消除Java SE 5，Java SE 6和Java EE环境之间的差异。

Spring还提供了集成类，用于支持与Timer相关的调度，自1.3以来的部分JDK以及Quartz Scheduler。这两个调度程序都使用FactoryBean进行设置，并分别选择引用Timer或Trigger实例，此外，Quartz Scheduler和Timer的一个方便类可用，可以调用现有目标对象的方法（类似于普通MethodInvokingFactoryBean操作）。

Spring的TaskExecutor接口与java.util.concurrent.Executor接口相同,事实上，存在的主要原因是在使用线程池时抽象出对Java 5的需求。

## TaskExecutor

### TaskExecutor接口定义

```java
public interface TaskExecutor extends Executor {

    @Override
    void execute(Runnable task);

    }
```

### TaskExecutor类图

![TaskExecutor类图](http://i1.ciimg.com/598959/ac1f02405f60c29c.png)

### Using a TaskExecutor

TaskExecutorExample

```java
import org.springframework.core.task.TaskExecutor;

public class TaskExecutorExample {

    private class MessagePrinterTask implements Runnable {

        private String message;

        public MessagePrinterTask(String message) {
            this.message = message;
        }

        public void run() {
            System.out.println(message);
        }

    }

    private TaskExecutor taskExecutor;

    public TaskExecutorExample(TaskExecutor taskExecutor) {
        this.taskExecutor = taskExecutor;
    }

    public void printMessages() {
        for(int i = 0; i < 25; i++) {
            taskExecutor.execute(new MessagePrinterTask("Message" + i));
        }
    }

}
```

xml文件配置

```xml
<bean id="taskExecutor" class="org.springframework.scheduling.concurrent.ThreadPoolTaskExecutor">
    <property name="corePoolSize" value="5" />
    <property name="maxPoolSize" value="10" />
    <property name="queueCapacity" value="25" />
</bean>

<bean id="taskExecutorExample" class="TaskExecutorExample">
    <constructor-arg ref="taskExecutor" />
</bean>
```

## TaskScheduler

### TaskScheduler接口定义

```java
public interface TaskScheduler {

    ScheduledFuture<?> schedule(Runnable task, Trigger trigger);


    ScheduledFuture<?> schedule(Runnable task, Date startTime);


    ScheduledFuture<?> scheduleAtFixedRate(Runnable task, Date startTime, long period);


    ScheduledFuture<?> scheduleAtFixedRate(Runnable task, long period);


    ScheduledFuture<?> scheduleWithFixedDelay(Runnable task, Date startTime, long delay);

    ScheduledFuture<?> scheduleWithFixedDelay(Runnable task, long delay);

}
```

### TaskScheduler类图

![TaskSchedulerl类图](http://i1.ciimg.com/598959/05d7c63caacfde74.png)

### Using a TaskScheduler

```xml
<task:scheduler id="scheduler" pool-size="10"/>

<task:scheduled-tasks>
    <task:scheduled ref="businessService" method="doSomething" cron="*/5 * * * * ?"/>
</task:scheduled-tasks>
```

## Using the Quartz Scheduler

Quartz使用Trigger，Job和JobDetail对象来实现各种作业的调度。为了方便起见，Spring提供了几个类，可简化Quartz在基于Spring的应用程序中的使用。

### Step 1: Configure Jobs in Quartz Scheduler

Spring提供了两种方式配置一个Job，如下：

* Using MethodInvokingJobDetailFactoryBean

    当您只需要调用某个特定bean的方法时，就非常方便。配置如下：

    ```xml
    <bean id="simpleJobDetail" class="org.springframework.scheduling.quartz.MethodInvokingJobDetailFactoryBean">
        <property name="targetObject" ref="businessObject" />
        <property name="targetMethod" value="doSomething" />
    </bean>
    ```

* Using JobDetailFactoryBean

    当您需要更高级的设置时，需要将数据传递给工作，更灵活。配置如下：

    ```xml
    <bean name="complexJobDetail" class="org.springframework.scheduling.quartz.JobDetailFactoryBean">
        <property name="jobClass" value="com.codersm.study.spring.scheduler.ScheduledJob"/>
        <property name="jobDataAsMap">
            <map>
                <entry key="timeout" value="5"/>
            </map>
        </property>
    </bean>
    ```

### Step 2: Configure Triggers to be used in Quartz Scheduler

* A: Simple Trigger , using SimpleTriggerFactoryBean

    ```xml
    <bean id="simpleTrigger"  class="org.springframework.scheduling.quartz.SimpleTriggerFactoryBean">
        <property name="jobDetail" ref="simpleJobDetail" />
        <property name="startDelay" value="1000" />
        <property name="repeatInterval" value="2000" />
    </bean>
    ```

* B: Cron Trigger , using CronTriggerFactoryBean

    ```xml
    <bean id="cronTrigger"  class="org.springframework.scheduling.quartz.CronTriggerFactoryBean">
        <property name="jobDetail" ref="complexJobDetail" />
        <property name="cronExpression" value="0/5 * * ? * SAT-SUN" />
    </bean>
    ```

### Step 3: Configure SchedulerFactoryBean that creates and configures Quartz Scheduler

```xml
<!-- Scheduler factory bean to glue together jobDetails and triggers to Configure Quartz Scheduler -->
<bean  class="org.springframework.scheduling.quartz.SchedulerFactoryBean">
    <property name="jobDetails">
        <list>
            <ref bean="simpleJobDetail" />
            <ref bean="complexJobDetail" />
        </list>
    </property>

    <property name="triggers">
        <list>
            <ref bean="simpleTrigger" />
            <ref bean="cronTrigger" />
        </list>
    </property>
</bean>
```