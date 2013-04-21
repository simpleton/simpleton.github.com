---
author: sunsj1231
comments: true
date: 2011-03-22 05:19:24
layout: post
slug: linux-kernel%e2%80%94%e2%80%94basic-knowledge
title: Linux kernel——basic knowledge
wordpress_id: 36001
categories:
- coding
---

the Linux Cross Reference:




[http://lxr.linux.no/](http://lxr.linux.no/)




This is the online codes which is easy to search and read.It is helpful when you want to read the kernel but there isnot offline linux kernel codes in you computer.










/proc/sys/kernel/printk




it can change the print level.




#define KERN_EMERG "<0>" /* system is unusable */  
#define KERN_ALERT "<1>" /* action must be taken immediately */  
#define KERN_CRIT "<2>" /* critical conditions */  
#define KERN_ERR "<3>" /* error conditions */  
#define KERN_WARNING "<4>" /* warning conditions */  
#define KERN_NOTICE "<5>" /* normal but significant condition */  
#define KERN_INFO "<6>" /* informational */  
#define KERN_DEBUG "<7>" /* debug-level messages */




All the information made by printk are saved in the log file which can be found by command "dmesg".




e.g. printk(KERN_EMERG "Hello World\n");
