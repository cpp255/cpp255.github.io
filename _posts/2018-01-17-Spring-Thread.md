---
layout: post
title:  "Spring 多线程"
date:   2018-01-17 20:05:35
tagline: "Supporting tagline"
tags : Java,Spring,Thread
published: true
categories: 源码阅读
---

# Spring 多线程
-------

​    Spring 的多线程开发的时候也会涉及到的，定时器、请求、异步任务等，不过很多都是Spring封装好的，拿来就用，最近项目里面用到了，就在 Spring 4.3.x 的版本上整理了相关的类。Spring 涉及到的类在好几个模块下：

1.基础类和接口定义在 spring-core 模块中，主要有：TaskExecutor、SyncTaskExecutor、AsyncTaskExecutor、SimpleAsyncTaskExecutor、TaskExecutorAdapter等。可以拿来即用，但是比较简单，一般无法满足我们的需求。其他模块相关的，都是继承和实现这个模块中的类。

2.定时器相关的类在 spring-context-support 中，有 scheduling.quartz, scheduling.commonj 2个包，另写一份文档。

3.spring-context 模块中的 scheduling 包下，有基础类的更完善的实现，比如：ThreadPoolExecutorFactoryBean、ThreadPoolTaskExecutor、ScheduledExecutorService等，这些类几乎可以满足我们的需求了。

**基础的TaskExecutor**

TaskExecutor 是最基础的任务执行接口，实现可以使用不同的执行策略：同步、异步、线程池等。跟jdk1.5的java.util.concurrent.Executor是相等的，Spring 3.0后是继承的 Executor。

**同步的SyncTaskExecutor**

主要用在测试场景，在许多场景中，推荐使用异步的 asynchronous

``` java
public class SyncTaskExecutor implements TaskExecutor, Serializable {
}
```



**异步的AsyncTaskExecutor**

实现 这个接口意味着 execute(Runnable)  方法，不会在调用线程中执行 Runnable，而是异步的在其他线程执行

```java
public interface AsyncTaskExecutor extends TaskExecutor {

	/** Constant that indicates immediate execution */
    // 立即执行
	long TIMEOUT_IMMEDIATE = 0;

	/** Constant that indicates no time limit */
   // 执行时间不受限制 
	long TIMEOUT_INDEFINITE = Long.MAX_VALUE;
}
```

**SimpleAsyncTaskExecutor**

实现了 TaskExecutor ，为每一个 task 生成一个新的线程，异步执行。支持通过bean的配置 "concurrencyLimit" 实现线程数的配置，默认线程数是无限的。使用于大量的短任务。

```java
@SuppressWarnings("serial")
public class SimpleAsyncTaskExecutor extends CustomizableThreadCreator
		implements AsyncListenableTaskExecutor, Serializable {
		/**
	    * execute 方法最终会调用该方法，默认如果没有实现 *threadFactory，则创建一个新的线程，并开始运行
	    **/
		protected void doExecute(Runnable task) {
		Thread thread = (this.threadFactory != null ? this.threadFactory.newThread(task) : createThread(task));
		thread.start();
	}
		}
```



**TaskExecutorAdapter **

适配器使用jdk的 Executor 暴露了 Spring 的 TaskExecutor，以此实现一个 TaskExecutor 运行.

```java
public class TaskExecutorAdapter implements AsyncListenableTaskExecutor {
  private final Executor concurrentExecutor;
	private TaskDecorator taskDecorator;
}
```



**ThreadPoolExecutorFactoryBean**

  实现了 FactoryBean 接口的类，通过 getObject() 方法可以获取到 ExecutorService 的实例，ExecutorService 的实现也是用的ThreadPoolExecutor。当然，这里面的 corePoolSize、maxPoolSize默认是1、 Integer.MAX_VALUE，不适合开发具体的场景的话，可以再修改参数。

```java
public class ThreadPoolExecutorFactoryBean extends ExecutorConfigurationSupport
		implements FactoryBean<ExecutorService>, InitializingBean, DisposableBean {
			private ExecutorService exposedExecutor;
}
```



**ThreadPoolTaskExecutor**

```java
public class ThreadPoolTaskExecutor extends ExecutorConfigurationSupport
		implements AsyncListenableTaskExecutor, SchedulingTaskExecutor {
	private ThreadPoolExecutor threadPoolExecutor;
	
	// 具体的ThreadPoolExecutor初始化
	@Override
	protected ExecutorService initializeExecutor(
			ThreadFactory threadFactory, RejectedExecutionHandler rejectedExecutionHandler) {

		BlockingQueue<Runnable> queue = createQueue(this.queueCapacity);

		ThreadPoolExecutor executor;
		if (this.taskDecorator != null) {
			executor = new ThreadPoolExecutor(
					this.corePoolSize, this.maxPoolSize, this.keepAliveSeconds, TimeUnit.SECONDS,
					queue, threadFactory, rejectedExecutionHandler) {
				@Override
				public void execute(Runnable command) {
					super.execute(taskDecorator.decorate(command));
				}
			};
		}
		else {
			executor = new ThreadPoolExecutor(
					this.corePoolSize, this.maxPoolSize, this.keepAliveSeconds, TimeUnit.SECONDS,
					queue, threadFactory, rejectedExecutionHandler);

		}

		if (this.allowCoreThreadTimeOut) {
			executor.allowCoreThreadTimeOut(true);
		}

		this.threadPoolExecutor = executor;
		return executor;
	}
}
```



**ThreadPoolTaskScheduler**

带调度功能的线程池配置，线程池实现是由 ScheduledExecutorService 的实现类 ScheduledThreadPoolExecutor，也有一些调度的方法。

```java
public class ThreadPoolTaskScheduler extends ExecutorConfigurationSupport
		implements AsyncListenableTaskExecutor, SchedulingTaskExecutor, TaskScheduler {
	private volatile ScheduledExecutorService scheduledExecutor;
	
	// 具体的ScheduledThreadPoolExecutor实例生成
	@Override
	protected ExecutorService initializeExecutor(
			ThreadFactory threadFactory, RejectedExecutionHandler rejectedExecutionHandler) {

		this.scheduledExecutor = createExecutor(this.poolSize, threadFactory, rejectedExecutionHandler);

		if (this.removeOnCancelPolicy) {
			if (setRemoveOnCancelPolicyAvailable && this.scheduledExecutor instanceof ScheduledThreadPoolExecutor) {
				((ScheduledThreadPoolExecutor) this.scheduledExecutor).setRemoveOnCancelPolicy(true);
			}
			else {
				logger.info("Could not apply remove-on-cancel policy - not a Java 7+ ScheduledThreadPoolExecutor");
			}
		}

		return this.scheduledExecutor;
	}
	
	protected ScheduledExecutorService createExecutor(
			int poolSize, ThreadFactory threadFactory, RejectedExecutionHandler rejectedExecutionHandler) {

		return new ScheduledThreadPoolExecutor(poolSize, threadFactory, rejectedExecutionHandler);
	}
}
```