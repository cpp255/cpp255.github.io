---
layout: post
title:  "Spring 定时器实现原理(2)"
date:   2018-01-22 19:47:10
tagline: "Supporting tagline"
tags : Java,Spring,quartz
published: true
categories: 源码阅读
---

# Spring 定时器实现原理(2)
-------

**QuartzScheduler**

    类说明里面是这么说的： Quartz 的心脏，间接实现了  org.quartz.Scheduler 的接口，包含了 schedule 调度 Job 的方法，JobListener 实例的注册等。 监听事件会在 scheduler 执行了不同的方法后，调用这些监听事件。RemotableQuartzScheduler 接口是继承的 java.rmi.Remote，声明了任务调度的方法。

```java
public class QuartzScheduler implements RemotableQuartzScheduler {
    // 定时器的监听管理和注册监听容器
    private ListenerManager listenerManager = new ListenerManagerImpl();
    // 这里不是很明白为什么要定为常量10，HashMap 中的数组一定是2的指数，10 肯定会变成大小为 16 
    private HashMap<String, JobListener> internalJobListeners = new HashMap<String, JobListener>(10);
    private HashMap<String, TriggerListener> internalTriggerListeners = new HashMap<String, TriggerListener>(10);
    private ArrayList<SchedulerListener> internalSchedulerListeners = new ArrayList<SchedulerListener>(10);
}
```



**QuartzSchedulerThread**

​    QuartzSchedulerThread 继承了 java 的 Thread 类，负责执行在 QuartzScheduler 中注册了 Trigger 的触发点的定时调用。这个线程只是执行触发器的定点计算和管理，并不会在这里面进行 Job 的运行，当然，这里会把符合到点的任务提交到 scheduler 线程池中运行。线程里面主要的类是 run() 方法，只要没有设置中断或者停机状态，里面的 while 循环是不停的运行，并默认30s，根据当前 scheduler 可用线程等状态，获取到点的可运行的 triggers 列表，再创建相应的任务，把跟 trigger 相关的 job 提交运行。



```java
public class QuartzSchedulerThread extends Thread {
  	// scheduler 携带的资源信息
	private QuartzSchedulerResources qsRsrcs;
  
/**
* 该线程会一直不停的循环执行(线程是正常运行状态，没有被停止)
**/
public void run() {
        int acquiresFailed = 0;

        while (!halted.get()) {
            try {
                // check if we're supposed to pause...
                synchronized (sigLock) {
                    while (paused && !halted.get()) {
                        try {
                            // wait until togglePause(false) is called...
                            sigLock.wait(1000L);
                        } catch (InterruptedException ignore) {
                        }

                        // reset failure counter when paused, so that we don't
                        // wait again after unpausing
                        acquiresFailed = 0;
                    }

                    if (halted.get()) {
                        break;
                    }
                }

                // wait a bit, if reading from job store is consistently
                // failing (e.g. DB is down or restarting)..
                if (acquiresFailed > 1) {
                    try {
                        long delay = computeDelayForRepeatedErrors(qRsrcs.getJobStore(), acquiresFailed);
                        Thread.sleep(delay);
                    } catch (Exception ignore) {
                    }
                }

                int availThreadCount = qsRsrcs.getThreadPool().blockForAvailableThreads();
                if(availThreadCount > 0) { // will always be true, due to semantics of blockForAvailableThreads...

                    List<OperableTrigger> triggers;

                    long now = System.currentTimeMillis();

                    clearSignaledSchedulingChange();
                    try {
                        // 获取从现在起，一定时间(默认30s)内，根据可用线程数等条件获取即将到来的触发器列表
                        triggers = qsRsrcs.getJobStore().acquireNextTriggers(
                                now + idleWaitTime, Math.min(availThreadCount, qsRsrcs.getMaxBatchSize()), qsRsrcs.getBatchTimeWindow());
                        acquiresFailed = 0;
                        if (log.isDebugEnabled())
                            log.debug("batch acquisition of " + (triggers == null ? 0 : triggers.size()) + " triggers");
                    } catch (JobPersistenceException jpe) {
                        if (acquiresFailed == 0) {
                            qs.notifySchedulerListenersError(
                                "An error occurred while scanning for the next triggers to fire.",
                                jpe);
                        }
                        if (acquiresFailed < Integer.MAX_VALUE)
                            acquiresFailed++;
                        continue;
                    } catch (RuntimeException e) {
                        if (acquiresFailed == 0) {
                            getLog().error("quartzSchedulerThreadLoop: RuntimeException "
                                    +e.getMessage(), e);
                        }
                        if (acquiresFailed < Integer.MAX_VALUE)
                            acquiresFailed++;
                        continue;
                    }

                    if (triggers != null && !triggers.isEmpty()) {

                        now = System.currentTimeMillis();
                        long triggerTime = triggers.get(0).getNextFireTime().getTime();
                        long timeUntilTrigger = triggerTime - now;
                        while(timeUntilTrigger > 2) {
                            synchronized (sigLock) {
                                if (halted.get()) {
                                    break;
                                }
                                if (!isCandidateNewTimeEarlierWithinReason(triggerTime, false)) {
                                    try {
                                        // we could have blocked a long while
                                        // on 'synchronize', so we must recompute
                                        now = System.currentTimeMillis();
                                        timeUntilTrigger = triggerTime - now;
                                        if(timeUntilTrigger >= 1)
                                            sigLock.wait(timeUntilTrigger);
                                    } catch (InterruptedException ignore) {
                                    }
                                }
                            }
                            if(releaseIfScheduleChangedSignificantly(triggers, triggerTime)) {
                                break;
                            }
                            now = System.currentTimeMillis();
                            timeUntilTrigger = triggerTime - now;
                        }

                        // this happens if releaseIfScheduleChangedSignificantly decided to release triggers
                        if(triggers.isEmpty())
                            continue;

                        // set triggers to 'executing'
                        List<TriggerFiredResult> bndles = new ArrayList<TriggerFiredResult>();

                        boolean goAhead = true;
                        synchronized(sigLock) {
                            goAhead = !halted.get();
                        }
                        if(goAhead) {
                            try {
                                // 通知这批触发器执行时间已到，调用数据库获取和修改相应的数据
                                List<TriggerFiredResult> res = qsRsrcs.getJobStore().triggersFired(triggers);
                                if(res != null)
                                    bndles = res;
                            } catch (SchedulerException se) {
                                qs.notifySchedulerListenersError(
                                        "An error occurred while firing triggers '"
                                                + triggers + "'", se);
                                //QTZ-179 : a problem occurred interacting with the triggers from the db
                                //we release them and loop again
                                for (int i = 0; i < triggers.size(); i++) {
                                    qsRsrcs.getJobStore().releaseAcquiredTrigger(triggers.get(i));
                                }
                                continue;
                            }

                        }

                        for (int i = 0; i < bndles.size(); i++) {
                            TriggerFiredResult result =  bndles.get(i);
                            TriggerFiredBundle bndle =  result.getTriggerFiredBundle();
                            Exception exception = result.getException();

                            if (exception instanceof RuntimeException) {
                                getLog().error("RuntimeException while firing trigger " + triggers.get(i), exception);
                                qsRsrcs.getJobStore().releaseAcquiredTrigger(triggers.get(i));
                                continue;
                            }

                            // it's possible to get 'null' if the triggers was paused,
                            // blocked, or other similar occurrences that prevent it being
                            // fired at this time...  or if the scheduler was shutdown (halted)
                            if (bndle == null) {
                                qsRsrcs.getJobStore().releaseAcquiredTrigger(triggers.get(i));
                                continue;
                            }

                            JobRunShell shell = null;
                            try {
                                // 创建 Job 执行的实例
                                shell = qsRsrcs.getJobRunShellFactory().createJobRunShell(bndle);
                                shell.initialize(qs);
                            } catch (SchedulerException se) {
                                qsRsrcs.getJobStore().triggeredJobComplete(triggers.get(i), bndle.getJobDetail(), CompletedExecutionInstruction.SET_ALL_JOB_TRIGGERS_ERROR);
                                continue;
                            }

                            // Job 的任务实例提交到 scheduler 中执行
                            if (qsRsrcs.getThreadPool().runInThread(shell) == false) {
                                // this case should never happen, as it is indicative of the
                                // scheduler being shutdown or a bug in the thread pool or
                                // a thread pool being used concurrently - which the docs
                                // say not to do...
                                getLog().error("ThreadPool.runInThread() return false!");
                                qsRsrcs.getJobStore().triggeredJobComplete(triggers.get(i), bndle.getJobDetail(), CompletedExecutionInstruction.SET_ALL_JOB_TRIGGERS_ERROR);
                            }

                        }

                        continue; // while (!halted)
                    }
                } else { // if(availThreadCount > 0)
                    // should never happen, if threadPool.blockForAvailableThreads() follows contract
                    continue; // while (!halted)
                }

                long now = System.currentTimeMillis();
                long waitTime = now + getRandomizedIdleWaitTime();
                long timeUntilContinue = waitTime - now;
                synchronized(sigLock) {
                    try {
                      if(!halted.get()) {
                        // QTZ-336 A job might have been completed in the mean time and we might have
                        // missed the scheduled changed signal by not waiting for the notify() yet
                        // Check that before waiting for too long in case this very job needs to be
                        // scheduled very soon
                        if (!isScheduleChanged()) {
                          sigLock.wait(timeUntilContinue);
                        }
                      }
                    } catch (InterruptedException ignore) {
                    }
                }

            } catch(RuntimeException re) {
                getLog().error("Runtime error occurred in main trigger firing loop.", re);
            }
        } // while (!halted)

        // drop references to scheduler stuff to aid garbage collection...
        qs = null;
        qsRsrcs = null;
    }
}
```



**QuartzSchedulerResources**

​	包含了创建 QuartzScheduler 实例必需的所有资源： JobStore、ThreadPool。



**JobRunShell**

​	JobRunShell 的实例负责提供安全的环境为 Job 运行，并且运行所有的 Job 的工作任务，捕抓人和抛出的异常，更新 Job 涉及的  Trigger。

​	JobRunShell 是通过工厂类 JobRunShellFactory 生成的实例，当 scheduler 判定一个 Job 已经被触发了，QuartzSchedulerThread 将会从配置的 ThreadPool 中的一个线程运行这个 shell。

​	JobRunShell 实现了 Runnable，因此 Job 的任务调用是在 run 的方法中运行。当然，这里也会涉及到注册事件的监听处理。

```java
public class JobRunShell extends SchedulerListenerSupport implements Runnable {
public void run() {
        qs.addInternalSchedulerListener(this);

        try {
            OperableTrigger trigger = (OperableTrigger) jec.getTrigger();
            JobDetail jobDetail = jec.getJobDetail();

            do {

                JobExecutionException jobExEx = null;
                Job job = jec.getJobInstance();

                try {
                    begin();
                } catch (SchedulerException se) {
                    qs.notifySchedulerListenersError("Error executing Job ("
                            + jec.getJobDetail().getKey()
                            + ": couldn't begin execution.", se);
                    break;
                }

                // notify job & trigger listeners...
                try {
                    if (!notifyListenersBeginning(jec)) {
                        break;
                    }
                } catch(VetoedException ve) {
                    try {
                        CompletedExecutionInstruction instCode = trigger.executionComplete(jec, null);
                        qs.notifyJobStoreJobVetoed(trigger, jobDetail, instCode);
                        
                        // QTZ-205
                        // Even if trigger got vetoed, we still needs to check to see if it's the trigger's finalized run or not.
                        if (jec.getTrigger().getNextFireTime() == null) {
                            qs.notifySchedulerListenersFinalized(jec.getTrigger());
                        }

                        complete(true);
                    } catch (SchedulerException se) {
                        qs.notifySchedulerListenersError("Error during veto of Job ("
                                + jec.getJobDetail().getKey()
                                + ": couldn't finalize execution.", se);
                    }
                    break;
                }

                long startTime = System.currentTimeMillis();
                long endTime = startTime;

                // execute the job 执行具体的任务
                try {
                    log.debug("Calling execute on job " + jobDetail.getKey());
                    job.execute(jec);
                    endTime = System.currentTimeMillis();
                } catch (JobExecutionException jee) {
                    endTime = System.currentTimeMillis();
                    jobExEx = jee;
                    getLog().info("Job " + jobDetail.getKey() +
                            " threw a JobExecutionException: ", jobExEx);
                } catch (Throwable e) {
                    endTime = System.currentTimeMillis();
                    getLog().error("Job " + jobDetail.getKey() +
                            " threw an unhandled Exception: ", e);
                    SchedulerException se = new SchedulerException(
                            "Job threw an unhandled exception.", e);
                    qs.notifySchedulerListenersError("Job ("
                            + jec.getJobDetail().getKey()
                            + " threw an exception.", se);
                    jobExEx = new JobExecutionException(se, false);
                }

                jec.setJobRunTime(endTime - startTime);

                // notify all job listeners
                if (!notifyJobListenersComplete(jec, jobExEx)) {
                    break;
                }

                CompletedExecutionInstruction instCode = CompletedExecutionInstruction.NOOP;

                // update the trigger
                try {
                    instCode = trigger.executionComplete(jec, jobExEx);
                } catch (Exception e) {
                    // If this happens, there's a bug in the trigger...
                    SchedulerException se = new SchedulerException(
                            "Trigger threw an unhandled exception.", e);
                    qs.notifySchedulerListenersError(
                            "Please report this error to the Quartz developers.",
                            se);
                }

                // notify all trigger listeners
                if (!notifyTriggerListenersComplete(jec, instCode)) {
                    break;
                }

                // update job/trigger or re-execute job
                if (instCode == CompletedExecutionInstruction.RE_EXECUTE_JOB) {
                    jec.incrementRefireCount();
                    try {
                        complete(false);
                    } catch (SchedulerException se) {
                        qs.notifySchedulerListenersError("Error executing Job ("
                                + jec.getJobDetail().getKey()
                                + ": couldn't finalize execution.", se);
                    }
                    continue;
                }

                try {
                    complete(true);
                } catch (SchedulerException se) {
                    qs.notifySchedulerListenersError("Error executing Job ("
                            + jec.getJobDetail().getKey()
                            + ": couldn't finalize execution.", se);
                    continue;
                }

                qs.notifyJobStoreJobComplete(trigger, jobDetail, instCode);
                break;
            } while (true);

        } finally {
            qs.removeInternalSchedulerListener(this);
        }
    }
}
```