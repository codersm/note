---
title: quartz学习
date: 2016-07-05T18:25:18.000Z
tags: 计划任务
---

任务调用组件 <!-- more -->

# quartz学习笔记之Job Stores

> 作业存储负责保存调度器工作数据的轨迹：工作、触发器、日历等，选择合适的作业存储对调度器来说是很重要。

QuartzScheduler.scheduleJob()方法

```java
public Date scheduleJob(JobDetail jobDetail,
            Trigger trigger) throws SchedulerException {

        //存储job和trigger
        resources.getJobStore().storeJobAndTrigger(jobDetail, trig);
        notifySchedulerListenersJobAdded(jobDetail);
        notifySchedulerThread(trigger.getNextFireTime().getTime());
        notifySchedulerListenersSchduled(trigger);
        return ft;
    }
```

JobStoreTX.storeJobAndTrigger()方法

```java
public void storeJobAndTrigger(final JobDetail newJob,
          final OperableTrigger newTrigger)
      throws JobPersistenceException {
      executeInLock(
          (isLockOnInsert()) ? LOCK_TRIGGER_ACCESS : null,
          new VoidTransactionCallback() {
              public void executeVoid(Connection conn) throws JobPersistenceException {
                  storeJob(conn, newJob, false);
                  storeTrigger(conn, newTrigger, newJob, false,
                          Constants.STATE_WAITING, false, false);
              }
          });
  }
```

storeTrigger()方法

```java
protected void storeTrigger(Connection conn,
        OperableTrigger newTrigger, JobDetail job, boolean replaceExisting, String state,
        boolean forceState, boolean recovering)
    throws JobPersistenceException {

    boolean existingTrigger = triggerExists(conn, newTrigger.getKey());

    if ((existingTrigger) && (!replaceExisting)) {
        throw new ObjectAlreadyExistsException(newTrigger);
    }

    try {

        boolean shouldBepaused;

        if (!forceState) {
            shouldBepaused = getDelegate().isTriggerGroupPaused(
                    conn, newTrigger.getKey().getGroup());

            if(!shouldBepaused) {
                shouldBepaused = getDelegate().isTriggerGroupPaused(conn,
                        ALL_GROUPS_PAUSED);

                if (shouldBepaused) {
                    getDelegate().insertPausedTriggerGroup(conn, newTrigger.getKey().getGroup());
                }
            }

            if (shouldBepaused && (state.equals(STATE_WAITING) || state.equals(STATE_ACQUIRED))) {
                state = STATE_PAUSED;
            }
        }

        if(job == null) {
            job = getDelegate().selectJobDetail(conn, newTrigger.getJobKey(), getClassLoadHelper());
        }
        if (job == null) {
            throw new JobPersistenceException("The job ("
                    + newTrigger.getJobKey()
                    + ") referenced by the trigger does not exist.");
        }

        if (job.isConcurrentExectionDisallowed() && !recovering) {
            state = checkBlockedState(conn, job.getKey(), state);
        }

        if (existingTrigger) {
            getDelegate().updateTrigger(conn, newTrigger, state, job);
        } else {
            getDelegate().insertTrigger(conn, newTrigger, state, job);
        }
    } catch (Exception e) {
        throw new JobPersistenceException("Couldn't store trigger '" + newTrigger.getKey() + "' for '"
                + newTrigger.getJobKey() + "' job:" + e.getMessage(), e);
    }
}
```

**1、RAMJobStore**

- 优点 配置简单，高性能。
- 缺点 应用程序结束或崩溃时，调度信息会丢失。

**2、JDBCJobStore**

**3、TerracottaJobStore**

**参考资料**： <http://www.tuicool.com/articles/AnyiAfa> <http://www.quartz-scheduler.org/documentation/quartz-2.2.x/tutorials/tutorial-lesson-09.html>
