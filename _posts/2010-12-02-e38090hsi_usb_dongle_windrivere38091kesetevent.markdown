---
author: sunsj1231
comments: true
date: 2010-12-02 06:47:06
layout: post
slug: '%e3%80%90hsi_usb_dongle_windriver%e3%80%91kesetevent'
title: 【HSI_USB_dongle_WINDriver】KeSetEvent
wordpress_id: 16001
categories:
- coding
---

最近在重构这个驱动，顺便复习了下windows driver。发现有很多细节以前学习的时候都没有注意。比如这个topic.....




**LONG **  
**KeSetEvent(**  
** IN PRKEVENT** _[Event](/MS.WDK.v10.7600.091201/Kernel_r/hh/Kernel_r/k105_0b9a87b5-bdf2-4449-81f6-1836ea47f038.xml.htm)_**,**  
** IN KPRIORITY** _[Increment](/MS.WDK.v10.7600.091201/Kernel_r/hh/Kernel_r/k105_0b9a87b5-bdf2-4449-81f6-1836ea47f038.xml.htm)_**,**  
** IN BOOLEAN** _[Wait](/MS.WDK.v10.7600.091201/Kernel_r/hh/Kernel_r/k105_0b9a87b5-bdf2-4449-81f6-1836ea47f038.xml.htm)_  
** );**




the parameter Wait always set be FALSE in my code.and I did not know why -_-




actually,If _Wait_ = TRUE, the routine returns without lowering the IRQL. In this  case, the **KeSetEvent** call must be immediately followed by a  **KeWait**_**Xxx**_ call. By setting _Wait_ = TRUE, the caller  can prevent an unnecessary context switch from occurring between the  **KeSetEvent** call and the **KeWait**_**Xxx**_ call. The  **KeWait**_**Xxx**_ routine, before it returns, restores the IRQL to  its original value at the start of the **KeSetEvent** call. Although the IRQL  disables context switches between the two calls, these calls cannot reliably be  used as the start and end of an atomic operation. For example, between these two  calls, a thread that is running at the same time on another processor might  change the state of the event object or of the target of the wait.(it is copied from wdk document).




wait参数的目的是允许在内部快速地把控制从一个线程传递到另一个线程。除了设备驱动程序之外，大部分系统部件都可以创建双事件对象。例如，客户线程和服务器线程使用双事件对象来界定它们的通信。当服务器线程需要唤醒对应的客户线程时，它首先调用KeSetEvent函数，并指定wait参数为TRUE，然后立即调用KeWaitXxx函数使自己进入睡眠状态。由于这两个操作都以原子方式完成，所以在控制交接时没有其它线程被唤醒。 DDK总是稍稍地描述一些内部细节，但我发现有些描述另人迷惑。我将以另一种方式解释这些内部细节，看过这些细节后你就会明白为什么我们总指定这个参数为FALSE。在内部，内核使用一个"同步数据库锁(dispatcher database lock)"来保护线程的阻塞、唤醒，和调度操作。KeSetEvent函数需要获取这个锁，KeWaitXxx函数也是这样。如果你把这个参数指定为TRUE，则KeSetEvent函数将设置一个标志以便KeWaitXxx函数知道你使用了TRUE参数，然后它返回，并且不释放这个锁。当你后来(应该立即调用，因为你此时正运行在一个比任何硬件设备都高的IRQL上，并且你占有着一个被极其频繁争夺的自旋锁)调用KeWaitXxx函数时，它不必再获取这个锁。产生的效果就是你唤醒了等待的线程并同时把自己置入睡眠状态，而不给其它线程任何运行的机会。 你应该明白，以wait参数为TRUE调用KeSetEvent的函数必须存在于非分页内存中，因为它在某段时间内执行在提升的IRQL上。很难想象一个普通设备驱动程序会需要使用这种机制，因为驱动程序决不会比内核更了解线程的调度。底线是：对该参数总使用FALSE。







TODO:understand what is "同步数据库锁(dispatcher database lock)"
