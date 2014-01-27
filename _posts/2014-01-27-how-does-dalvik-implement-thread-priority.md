---
layout: post
title: "How does Dalvik implement Thread Priority"
description: ""
category: Thread
tags: [android]
---
{% include JB/setup %}

# Java Thread Implementation#

On linux platform, jvm implement thread in native thread method, every jvm thread will map a native thread. And so on, native thread under the linux platform are implemented as processes that shared resources. The scheduler does not differentiate between a thread and a process.
All threads of all apps belong on the same thread group on the system, so every thread competes with all threads of all apps. Therefore, `Thread.setPriority` should actually do the same as `Process.setThreadPriority`.

Currently, CFS is the default scheduler for process or thread on linux platform. And also there is other schedule for real time task.

# CFS Priority Scheduling#
**SCHED_NORMAL**, **SCHED_BATCH**, **SCHED_IDEL** will be scheduled by CFS scheduler.

**SCHED_RR** and **SCHED_FIFO** will be scheduled by real time scheduler.

# Android Thread Pirority Implementation#

<http://androidxref.com/4.2_r1/xref/dalvik/vm/os/android.cpp#48>
{% highlight java %}

	void os_changeThreadPriority(Thread* thread, int newPriority)
	{
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

{% endhighlight %}

<http://androidxref.com/4.2_r1/xref/system/core/libcutils/sched_policy.c#312>

{% highlight java %}

	if (__sys_supports_schedgroups) {
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
{% endhighlight %}


Above mentioned source code tell us, the different thread pirority will map to `SP_FORCEGROUND` and `SP_BACKGROUND` priority. So invoking the `Thread.setPriority` only make few sense.

-----
# References#
[1]: <http://lwn.net/Articles/3866/>
[2]: <http://androidxref.com/source/xref/dalvik/vm/Thread.cpp>
[3]: <http://stackoverflow.com/questions/1662185/do-linux-jvms-actually-implement-thread-priorities>


