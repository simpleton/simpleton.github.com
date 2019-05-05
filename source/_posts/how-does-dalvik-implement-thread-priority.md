---
layout: post
title: How does Dalvik implement Thread Priority
date: 01-27-2014 00:00:00
tags:
  - android
	- thread
---

# Java Thread Implementation#

On linux platform, jvm implement thread in native thread method, every jvm thread will map a native thread. And so on, native thread under the linux platform are implemented as processes that shared resources. The scheduler does not differentiate between a thread and a process.
All threads of all apps belong on the same thread group on the system, so every thread competes with all threads of all apps. Therefore, `Thread.setPriority` should actually do the same as `Process.setThreadPriority`.

Currently, CFS is the default scheduler for process or thread on linux platform. And also there is other schedule for real time task.

# CFS Priority Scheduling#
**SCHED_NORMAL**, **SCHED_BATCH**, **SCHED_IDEL** will be scheduled by CFS scheduler.

**SCHED_RR** and **SCHED_FIFO** will be scheduled by real time scheduler.

In Complete Fair Scheduler every task has the virtual time parameter to save the time that it had token cup time. And the system record the fair clock which is running in RealTime div TheNumberOfTask. Every task's vtime will trbby to catch the fair clock. All task will save in RB tree with its vtime. In every sechuldeing happened, the sechudler will choose the left of RB tree which is has the most small vtime.

On the other hand, every task has seperated priority. If one task has two times priority than the other, the high priority task will get two times cup time than the other.
# Android Thread Pirority Implementation#

![Priority](http://pic002.cnblogs.com/images/2011/261337/2011110610315226.png)

<http://androidxref.com/4.2_r1/xref/dalvik/vm/os/android.cpp#48>
```c++
void os_changeThreadPriority(Thread* thread, int newPriority)	{
		if (newPriority < 1 || newPriority > 10) {
		ALOGW("bad priority %d", newPriority);
        newPriority = 5;
    }

    int newNice = kNiceValues[newPriority-1];
    pid_t pid = thread->systemTid;

    if (newNice >= ANDROID_PRIORITY_BACKGROUND) {
        set_sched_policy(dvmGetSysThreadId(), SP_BACKGROUND);
    } else if (getpriority(PRIO_PROCESS, pid) >= ANDROID_PRIORITY_BACKGROUND) {
        set_sched_policy(dvmGetSysThreadId(), SP_FOREGROUND);
    }

    if (setpriority(PRIO_PROCESS, pid, newNice) != 0) {
        std::string threadName(dvmGetThreadName(thread));
        ALOGI("setPriority(%d) '%s' to prio=%d(n=%d) failed: %s",
        pid, threadName.c_str(), newPriority, newNice, strerror(errno));
    } else {
        ALOGV("setPriority(%d) to prio=%d(n=%d)", pid, newPriority, newNice);
    }
}
```

<http://androidxref.com/4.2_r1/xref/system/core/libcutils/sched_policy.c#312>

```c++
  if (sys_supports_schedgroups) {
      if (add_tid_to_cgroup(tid, policy)) {
            if (errno != ESRCH && errno != ENOENT)
                return -errno;
			}
	} else {
        struct sched_param param;

        param.sched_priority = 0;
        sched_setscheduler(tid,
                           (policy == SP_BACKGROUND) ?
                            SCHED_BATCH : SCHED_NORMAL,
                           &param);
  }
```


<http://lxr.linux.no/#linux+v3.13.5/kernel/sched/core.c#L3015>

```C++
	/* Actually do priority change: must hold rq lock. */
	static void
	__setscheduler(struct rq *rq, struct task_struct *p, int policy, int prio)
	{
		p->policy = policy;
		p->rt_priority = prio;
		p->normal_prio = normal_prio(p);
		/* we are holding p->pi_lock already */
		p->prio = rt_mutex_getprio(p);
		if (rt_prio(p->prio))
			p->sched_class = &rt_sched_class;
		else
			p->sched_class = &fair_sched_class;
		set_load_weight(p);
	}
```

<http://lxr.linux.no/#linux+v3.13.5/kernel/sched/core.c#L2462>

```C++
	static inline struct task_struct *
	pick_next_task(struct rq *rq)
	{
        const struct sched_class *class;
        struct task_struct *p;

        /*
         * Optimization: we know that if all tasks are in
         * the fair class we can call that function directly:
         */
        if (likely(rq->nr_running == rq->cfs.h_nr_running)) {
                p = fair_sched_class.pick_next_task(rq);
                if (likely(p))
                        return p;
        }

        for_each_class(class) {
                p = class->pick_next_task(rq);
                if (p)
                        return p;
        }

        BUG(); /* the idle class will always have a runnable task */
	}
```


# Demo Time#

<https://github.com/simpleton/android_misc/tree/master/ThreadBenchmark>

# References#

1. <http://lwn.net/Articles/3866/>
2. <http://androidxref.com/source/xref/dalvik/vm/Thread.cpp>
3. <http://stackoverflow.com/questions/1662185/do-linux-jvms-actually-implement-thread-priorities>
4. <http://blog.csdn.net/innost/article/details/6940136>
5. <https://www.ibm.com/developerworks/cn/linux/l-cfs/>
6. <http://hi.baidu.com/_kouu/item/055bd19af9f6b9dc1f4271d1>
7. <http://ck.wikia.com/wiki/SchedulingPolicies>
