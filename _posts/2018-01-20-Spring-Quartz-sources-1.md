---
layout: post
title:  "Spring 定时器实现原理(1)"
date:   2018-01-20 12:03:10
tagline: "Supporting tagline"
tags : Java,Spring,quartz
published: true
categories: 源码阅读
---

# Spring 定时器实现原理(1)
-------

​    定时器是常用到的功能之一，底层也是使用了多线程的任务调度，然后封装了更多的功能。

**Trigger**

​    触发器----任务被调度执行的触发规则。描述了决定下一次关联的任务执行时间的最基本触发器通用接口。CronTrigger 基于它实现，CronTrigger 允许定义到天相关、周相关、月相关、年相关的时间，3.x 的版本前还还有 CronTriggerBean 继续继承 CronTrigger，使得任务的使用更简单，4.3.x 之后已经删除该类，使用 CronTriggerFactoryBean 进行扩展，可定义的参数比之前的更丰富。类似的，该类也实现了 FactoryBean 的接口。类似简单的用法还有 SimpleTrigger。

**JobDetail**

​    作业详情——定义了被调度任务。 JobDetail 实现 了 org.quartz.Job jobDetail 接口，JobDetail 实例后注册到 Scheduler 中。 



```java
public class CronTriggerFactoryBean implements FactoryBean<CronTrigger>, BeanNameAware, InitializingBean {
  	private CronTrigger cronTrigger;
  
  @Override
	public void afterPropertiesSet() throws ParseException {
		if (this.name == null) {
			this.name = this.beanName;
		}
		if (this.group == null) {
			this.group = Scheduler.DEFAULT_GROUP;
		}
		if (this.jobDetail != null) {
			this.jobDataMap.put("jobDetail", this.jobDetail);
		}
		if (this.startDelay > 0 || this.startTime == null) {
			this.startTime = new Date(System.currentTimeMillis() + this.startDelay);
		}
		if (this.timeZone == null) {
			this.timeZone = TimeZone.getDefault();
		}
		
        // cronTrigger 实例使用了 CronTriggerImpl 生成
		CronTriggerImpl cti = new CronTriggerImpl();
		cti.setName(this.name);
		cti.setGroup(this.group);
		if (this.jobDetail != null) {
			cti.setJobKey(this.jobDetail.getKey());
		}
		cti.setJobDataMap(this.jobDataMap);
		cti.setStartTime(this.startTime);
		cti.setCronExpression(this.cronExpression);
		cti.setTimeZone(this.timeZone);
		cti.setCalendarName(this.calendarName);
		cti.setPriority(this.priority);
		cti.setMisfireInstruction(this.misfireInstruction);
		cti.setDescription(this.description);
		this.cronTrigger = cti;
	}
}
```



​    CronTrigger 实现了 Trigger 的接口，

```java
public class CronTrigger implements Trigger {

	private final CronSequenceGenerator sequenceGenerator;
}
```



​    CronSequenceGenerator 里对定时器的运行规则做了定义，expression 模式用六个空格分割的域，从左到右分别是：秒 分 时 日 月 周。月和周可以使用英文的前三个字母表示。比如 "0 0 8-10 * * *"  代表每天 8、9、10点。expression 会被拆解到具体的时间上，这里用了 BitSet 数据结构存储时间，类的构造器里面，通过传进来的 expression 参数，进行分解，然后根据定义的规则进行定义，就可以获取到具体的时间。

```java
public class CronSequenceGenerator {
	// 定时器重复的模式
	private final String expression;

	private final TimeZone timeZone;

	private final BitSet months = new BitSet(12);

	private final BitSet daysOfMonth = new BitSet(31);

	private final BitSet daysOfWeek = new BitSet(7);

	private final BitSet hours = new BitSet(24);

	private final BitSet minutes = new BitSet(60);

	private final BitSet seconds = new BitSet(60);
  // 被构造器调用的解析方法
  private void parse(String expression) throws IllegalArgumentException {
    // 空格拆解为6个时间域
		String[] fields = StringUtils.tokenizeToStringArray(expression, " ");
		if (!areValidCronFields(fields)) {
			throw new IllegalArgumentException(String.format(
					"Cron expression must consist of 6 fields (found %d in \"%s\")", fields.length, expression));
		}
		doParse(fields);
	}
  
  // 分别对6个域拆解分析
  private void doParse(String[] fields) {
		setNumberHits(this.seconds, fields[0], 0, 60);
		setNumberHits(this.minutes, fields[1], 0, 60);
		setNumberHits(this.hours, fields[2], 0, 24);
		setDaysOfMonth(this.daysOfMonth, fields[3]);
       	// 月和周因为是用英文的前三个字母描述，所以得把字母替换成数字再处理
		setMonths(this.months, fields[4]);
		setDays(this.daysOfWeek, replaceOrdinals(fields[5], "SUN,MON,TUE,WED,THU,FRI,SAT"), 8);

		if (this.daysOfWeek.get(7)) {
			// Sunday can be represented as 0 or 7
			this.daysOfWeek.set(0);
			this.daysOfWeek.clear(7);
		}
	}
  
  // 对单独的一个时间区域进行规则匹配分解
  private void setNumberHits(BitSet bits, String value, int min, int max) {
		String[] fields = StringUtils.delimitedListToStringArray(value, ",");
		for (String field : fields) {
			if (!field.contains("/")) { // 单独一个值
				// Not an incrementer so it must be a range (possibly empty)
				int[] range = getRange(field, min, max);
				bits.set(range[0], range[1] + 1);
			}
			else { // 类似 "0/30"
				String[] split = StringUtils.delimitedListToStringArray(field, "/");
				if (split.length > 2) {
					throw new IllegalArgumentException("Incrementer has more than two fields: '" +
							field + "' in expression \"" + this.expression + "\"");
				}
				int[] range = getRange(split[0], min, max);
				if (!split[0].contains("-")) {
					range[1] = max - 1;
				}
				int delta = Integer.valueOf(split[1]);
				if (delta <= 0) {
					throw new IllegalArgumentException("Incrementer delta must be 1 or higher: '" +
							field + "' in expression \"" + this.expression + "\"");
				}
				for (int i = range[0]; i <= range[1]; i += delta) {
					bits.set(i);
				}
			}
		}
	}

  // 对一个域里面拆出来的一个值继续分解
	private int[] getRange(String field, int min, int max) {
		int[] result = new int[2];
		if (field.contains("*")) { // 没有设置，则是这个区间所有的值都被包括
			result[0] = min;
			result[1] = max - 1;
			return result;
		}
		if (!field.contains("-")) { // 单值数字的时候，区间值都是同一个数
			result[0] = result[1] = Integer.valueOf(field);
		}
		else { // 类似 "8-10"
			String[] split = StringUtils.delimitedListToStringArray(field, "-");
			if (split.length > 2) {
				throw new IllegalArgumentException("Range has more than two fields: '" +
						field + "' in expression \"" + this.expression + "\"");
			}
			result[0] = Integer.valueOf(split[0]);
			result[1] = Integer.valueOf(split[1]);
		}
		if (result[0] >= max || result[1] >= max) {
			throw new IllegalArgumentException("Range exceeds maximum (" + max + "): '" +
					field + "' in expression \"" + this.expression + "\"");
		}
		if (result[0] < min || result[1] < min) {
			throw new IllegalArgumentException("Range less than minimum (" + min + "): '" +
					field + "' in expression \"" + this.expression + "\"");
		}
		if (result[0] > result[1]) {
			throw new IllegalArgumentException("Invalid inverted range: '" + field +
					"' in expression \"" + this.expression + "\"");
		}
		return result;
	}
}
```



**MethodInvokingJobDetailFactoryBean**

​    具体的任务执行 bean 由 之前 3.2.x 版本的 BeanInvokingJobDetailFactoryBean 替换成了 MethodInvokingJobDetailFactoryBean，可配置项也更加丰富。可以指定具体的被调度的类，和被调度类的实例的方法。该类也同样的实现了 FactoryBean 接口。如下是定义了串行的运行 ID 为 testSkuExecutor 实例的 execute 方法.

```xml
<property name="concurrent" value="false"/>
		<property name="targetBean" value="testSkuExecutor"/>
		<property name="targetMethod" value="execute"/>
```



**SchedulerFactoryBean**

​    在调度任务中执行持久化操作时，强烈推荐使用  Spring 管理或者 JTA 的事务方式。否则，数据库锁将不能正常的工作和可能被破坏。

```java
public class SchedulerFactoryBean extends SchedulerAccessor implements FactoryBean<Scheduler>,
		BeanNameAware, ApplicationContextAware, InitializingBean, DisposableBean, SmartLifecycle {
    // 父类 SchedulerAccessor 中定义的数据结构，用来保存触发器列表的id值
    private List<Trigger> triggers;
	private DataSource dataSource;
}
```



**Scheduler**

​    org.quartz 包中的 Scheduler 维护了 JobDetail 和 Triger 的注册。注册后，一旦关联的 Trigger 到触发点， Scheduler 会负责相应的 Job 执行。SchedulerFactory 负责生成 Scheduler 的实例。Scheduler 创建后必须执行 start() 方法才可以运行 Job。scheduleJob(JobDetail, Trigger) 或者 addJob(JobDetail, boolean) 方法用来注册 JobDetail 的实例。

   Job 和 Trigger 会被命名并且加入到 group，group 用来创建逻辑的组或者有关 Job 和 Triggers 的类目，默认可以使用名字为常量的 DEFAULT_GROUP group。

​    程序提供了监听 JobListener 接口实现监听相关的事件。TriggerListener 提供了 Trigger 触发点的运行，SchedulerListener 提供了 Scheduler 运行的事件和错误。使用 ListenerManager 添加监听事件。



```java
public interface Scheduler {
  Date scheduleJob(JobDetail jobDetail, Trigger trigger)
        throws SchedulerException;
  
  // 添加 JobDetail 且没有 Trigger 关联，Job 将会被搁置，直至用一个 Trigger 进行调度，或者被 cheduler.triggerJob() 调用
  void addJob(JobDetail jobDetail, boolean replace)
        throws SchedulerException;
}
```



​	Spring 3.2.x 下的定时任务使用：

```xml
<bean id="scheduler" class="org.springframework.scheduling.quartz.SchedulerFactoryBean">
		<property name="configLocation" value="classpath:quartz.properties" />
		<property name="dataSource" ref="dataSource" />
		<property name="overwriteExistingJobs" value="true" />
        <!-- 被调度的 Trigger 列表 -->
		<property name="triggers">
			<list>
				<ref bean="testExecutorCronTrigger" />
			</list>
		</property>
	</bean>

    <!-- 被调度的 JobDetail -->
	<bean id="testExecutorJobDetail" class="frameworkx.springframework.scheduling.quartz.BeanInvokingJobDetailFactoryBean">
        <!-- 具体调度任务执行的类id -->
		<property name="targetBean" value="testExecutor" />
        <!-- 具体调度任务执行的类的方法 -->
		<property name="targetMethod" value="execute" />
        <!-- 是否允许并行执行 -->
		<property name="concurrent" value="false" />
	</bean>

    <!-- 被调度的 Trigger -->
	<bean id="testExecutorCronTrigger" class="org.springframework.scheduling.quartz.CronTriggerBean">
        <!-- 相关的 JobDetail  -->
		<property name="jobDetail" ref="testExecutorJobDetail" />
        <!-- 触发器的时间定义：频率为每2分钟一次 -->
		<property name="cronExpression" value="0 */2 * * * ?" />
	</bean>
```

