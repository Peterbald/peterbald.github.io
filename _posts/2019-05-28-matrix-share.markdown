---
layout: post
title: "Matrix 监控方案解析"
subtitle: ""
description: "Matrix, 监控, 源码解析"
author: "RandomJ"
header-img: "img/pragserWildsee.jpg"
tags:
  - Matrix
  - iOS
  - 监控
  - 源码解析
---

#### 一、概述

最近在做监控相关的东西，刚好趁这个机会剖析下腾讯开源的 Matrix 的实现。

#### 二、Matrix 简介

> Matrix 是一款微信团队研发并日常使用的应用性能接入框架，支持 iOS, macOS 和 Android。 Matrix 通过接入各种性能监控方案，对性能监控项的异常数据进行采集和分析，输出相应的问题分析、定位与优化建议，从而帮助开发者开发出更高质量的应用。

Matrix 架构主要采用插件模式

- Matrix 主要负责插件的启动、停止等操作
- MatrixBuilder 负责持有插件，并向外部提供插件生命周期的统一回调
- MatrixPlugin 插件的抽象类

![image-20190523163859121](http://ww3.sinaimg.cn/large/006tNc79gy1g3bc9eoh4sj30mp0ehwh6.jpg)

#### 三、Crash 捕获

Crash Free 是我们平时关注比较多的稳定性指标，在了解 Crash 监控之前，先来了解 iOS 平台 Crash 产生的原理

##### 3.1 OSX、iOS 的系统架构

![image-20190527115401277](http://ww4.sinaimg.cn/large/006tNc79gy1g3fqi4vscrj308g05gjri.jpg)

OSX 和 iOS 下系统架构主要分为四层：

- 应用层：也叫用户体验层，MacOS 平台 GUI 是 Aqua，iOS 平台是 SpringBoard。SpringBoard 主要是用来管理 iOS 设备上的主屏幕、图标等
- 应用框架层：MacOS 平台是 Cocoa 框架，包含 Foudation 和 AppKit 框架。iOS 平台是 Cocoa Touch 框架，包含 Foudation 和 UIKit 框架
- 核心框架层：也叫图形和媒体层。包括核心框架、Open GL 和 QuickTime
- Darwin：包含开放源代码的[XNU](https://zh.wikipedia.org/wiki/XNU)[内核](https://zh.wikipedia.org/wiki/内核)，其以[微核心](https://zh.wikipedia.org/wiki/微核心)为基础的核心架构来实现[Mach](https://zh.wikipedia.org/wiki/Mach)，而操作系统的服务和[用户空间](https://zh.wikipedia.org/wiki/使用者空間)工具则以[BSD](https://zh.wikipedia.org/wiki/BSD)为基础

在这里面重点理解清楚几个概念：

![image-20190527163329987](http://ww1.sinaimg.cn/large/006tNc79gy1g3fykxra0lj30du0csgqb.jpg)

- POSIX：表示可移植操作系统接口（Portable Operating System Interface of UNIX，缩写为 POSIX），POSIX 标准定义了操作系统应该为应用程序提供的接口标准，是针对 UNIX 提供的标准
- BSD：伯克利软件包(Berkeley Software Distribution，BSD)，是一个派生自 Unix 的操作系统。内核的 BSD 部分提供了 POSIX 应用程序接口
- Mach：由卡内基梅隆大学开发的计算机操作系统微内核。Mach 主要是作为传统 UNIX 内核的替代品出现的，引入了 port 的概念来表示双向的 IPC。Mach 中有以下几个概念
  - 任务：即拥有一组系统资源的对象，允许“线程”在其中执行
  - 线程：是执行的基本单位，拥有一个任务的上下文，并且共享任务中的资源
  - port：任务间通讯的一组受保护的消息队列，任务可以向任何 port 发送或接收数据
  - 消息：某些有类型的数据对象的集合，只可以发送至 port，而非某特定任务或者线程

总结：XNU 内核是一种混合式核心，兼具宏内核 BSD 和微内核 Mach

##### 3.2 iOS 系统的异常类型

- Mach 异常：用户态下可以直接通过 Mach API 设置 Thread/Task/Host 的异常端口来捕获异常。Mach 异常的定义如下

  ![image-20190527171954030](http://ww3.sinaimg.cn/large/006tNc79gy1g3fzx5rsg6j30cu0f6dj7.jpg)

- Unix 信号：维基百科中的定义如下

  > 信号（英语：Signals）是 Unix、类 Unix 以及其他 POSIX 兼容的操作系统中进程间通讯的一种有限制的方式。它是一种异步的通知机制，用来提醒进程一个事件已经发生。当一个信号发送给一个进程，操作系统中断了进程正常的控制流程，此时，任何非原子操作都将被中断。如果进程定义了信号的处理函数，那么它将被执行，否则就执行默认的处理函数。

  如果开发者没有捕获 Mach 异常，会被 host 层上的 ux_exception() 捕获到 Mach 异常，将之转换成 Unix Signal，通过 ThreadSignal 将信号投递到出错的线程，可以通过 signalHandler 回调来获取信号。Unix 信号定义如下：

  ![image-20190527172018469](http://ww1.sinaimg.cn/large/006tNc79gy1g3h2dod26uj30ds0fwwka.jpg)

异常的处理流程中，先由 Mach 内核抛出，通过 ux_exception() 转成 Unix Signal，所以处理 Crash 的时候应该优先处理 Mach 异常，如果 Mach 异常处理完成之后直接退出，Unix 信号不会有机会处理

##### 3.3 Matrix Crash 处理

- 基于 KSCrash

- Mach 异常处理：在 `KSCrashMonitor_MachException.c` 中的注册 handler 处理 Mach 异常

  ```c
  static bool installExceptionHandler() {
      bool attributes_created = false;
      pthread_attr_t attr;

      kern_return_t kr;
      int error;
      // 1.获取调用线程的任务端口
      const task_t thisTask = mach_task_self();
      exception_mask_t mask = EXC_MASK_BAD_ACCESS | EXC_MASK_BAD_INSTRUCTION | EXC_MASK_ARITHMETIC | EXC_MASK_SOFTWARE | EXC_MASK_BREAKPOINT;

      // 2.保存之前的异常端口号
      kr = task_get_exception_ports(thisTask, mask, g_previousExceptionPorts.masks, &g_previousExceptionPorts.count, g_previousExceptionPorts.ports, g_previousExceptionPorts.behaviors, g_previousExceptionPorts.flavors);

      if(g_exceptionPort == MACH_PORT_NULL) {
          // 3.创建异常处理端口
          kr = mach_port_allocate(thisTask, MACH_PORT_RIGHT_RECEIVE, &g_exceptionPort);

          // 4.给端口配置发送权限
          kr = mach_port_insert_right(thisTask, g_exceptionPort, g_exceptionPort, MACH_MSG_TYPE_MAKE_SEND);
      }

      // 5.设置异常处理端口
      kr = task_set_exception_ports(thisTask, mask, g_exceptionPort, EXCEPTION_DEFAULT, THREAD_STATE_NONE);

      pthread_attr_init(&attr);
      attributes_created = true;
      pthread_attr_setdetachstate(&attr, PTHREAD_CREATE_DETACHED);
      error = pthread_create(&g_secondaryPThread, &attr, &handleExceptions, kThreadSecondary);

      g_secondaryMachThread = pthread_mach_thread_np(g_secondaryPThread);
      ksmc_addReservedThread(g_secondaryMachThread);

      // 6.创建线程等待异常发送
      error = pthread_create(&g_primaryPThread, &attr, &handleExceptions, kThreadPrimary);
      pthread_attr_destroy(&attr);
      g_primaryMachThread = pthread_mach_thread_np(g_primaryPThread);
      ksmc_addReservedThread(g_primaryMachThread);

      return true;
  }
  ```

- Unix Signal 处理：

  ```c
  // Note: Dereferencing a NULL pointer causes SIGILL, ILL_ILLOPC on i386
  //       but causes SIGTRAP, 0 on arm.
  static const int g_fatalSignals[] = {
      SIGABRT,
      SIGBUS,
      SIGFPE,
      SIGILL,
      SIGPIPE,
      SIGSEGV,
      SIGSYS,
      SIGTRAP,
  };

  static bool installSignalHandler() {
    	// 1.获取异常信号数组
      const int* fatalSignals = kssignal_fatalSignals();
      int fatalSignalsCount = kssignal_numFatalSignals();

      if(g_previousSignalHandlers == NULL)
      {
  				// 2.保存之前的信号处理句柄
          g_previousSignalHandlers = malloc(sizeof(*g_previousSignalHandlers)
                                            * (unsigned)fatalSignalsCount);
      }

      // 3.构造 sigaction
      struct sigaction action = {{0}};
      action.sa_flags = SA_SIGINFO | SA_ONSTACK;
  #if KSCRASH_HOST_APPLE && defined(__LP64__)
      action.sa_flags |= SA_64REGSET;
  #endif
      sigemptyset(&action.sa_mask);
      action.sa_sigaction = &handleSignal;

    	// 4.遍历异常信号数组设置回调
      for(int i = 0; i < fatalSignalsCount; i++)
      {
          if(sigaction(fatalSignals[i], &action, &g_previousSignalHandlers[i]) != 0)
          {
              char sigNameBuff[30];
              const char* sigName = kssignal_signalName(fatalSignals[i]);
              if(sigName == NULL)
              {
                  snprintf(sigNameBuff, sizeof(sigNameBuff), "%d", fatalSignals[i]);
                  sigName = sigNameBuff;
              }
              KSLOG_ERROR("sigaction (%s): %s", sigName, strerror(errno));
              // Try to reverse the damage
              for(i--;i >= 0; i--) {
                  sigaction(fatalSignals[i], &g_previousSignalHandlers[i], NULL);
              }
              goto failed;
          }
      }
      return true;
  }
  ```

- C++ 异常处理：`std::set_terminate` 设置异常处理回调

- OC 异常处理：`NSSetUncaughtExceptionHandler` 设置异常处理回调

总结：在 Crash 的捕获这一部分，主要明确 iOS 平台的异常处理需要优先处理 Mach 内核异常，再处理 Unix Signal。Mach 的异常处理是通过在子线程使用 mach_port 读取异常端口消息的方式来获取异常消息。

#### 四、卡顿监测

Matrix 中的卡顿监控是基于 `Runloop` 来做的，先来了解下什么是 `Runloop`。

##### 4.1 Runloop

在阅读 [MrPeak](http://mrpeak.cn/blog/ios-runloop/) 大佬的文章和 `Runloop` 代码之后，总结如下图：

![](http://ww1.sinaimg.cn/large/006tNc79gy1g3g577c2slj30lz0dmq5s.jpg)

- doBlocks：调用 runloop 中 \_blocks_item 链表中的每个 block。通过 CFRunLoopPerformBlock 函数可以添加 block。
- doSource0：处理 runloopMode 相关的 source0，调用 runloopSource 的 perform 函数，参数被 runloopSource 持有。source0 有开发者可以调用的 API。
- doSource1：处理 runloopMode 相关的 source1，调用 runloopSource 的 perform 函数，参数被 runloopSource 持有。Source1 只用于执行系统任务。
- doTimers：处理 runloopMode 相关的 timers，遍历调用 `__CFRunLoopDoTimer` 函数。
- dispatch_main_queue_callback_4CF：处理通过 GCD 派发到 MainQueue 的任务。

总结下来，一个 runloop 循环中主要执行任务的时间有两段。一是第一次进入 runloop 循环开始处理 doTimers 和 doSources 事件到休眠，二是被再次唤醒之后到休眠。

##### 4.2 Matrix 实现

主线程上直接添加监听，在对应事件 `Observer` 回调时进行打点。

```objc
- (void)addRunLoopObserver {
    NSRunLoop *curRunLoop = [NSRunLoop currentRunLoop];
    // the first observer
    CFRunLoopObserverContext context = {0, (__bridge void *) self, NULL, NULL, NULL};
    CFRunLoopObserverRef beginObserver = CFRunLoopObserverCreate(kCFAllocatorDefault, kCFRunLoopAllActivities, YES, LONG_MIN, &myRunLoopBeginCallback, &context);
    CFRetain(beginObserver);
    m_runLoopBeginObserver = beginObserver;
    // the last observer
    CFRunLoopObserverRef endObserver = CFRunLoopObserverCreate(kCFAllocatorDefault, kCFRunLoopAllActivities, YES, LONG_MAX, &myRunLoopEndCallback, &context);
    CFRetain(endObserver);
    m_runLoopEndObserver = endObserver;
    CFRunLoopRef runloop = [curRunLoop getCFRunLoop];
    CFRunLoopAddObserver(runloop, beginObserver, kCFRunLoopCommonModes);
    CFRunLoopAddObserver(runloop, endObserver, kCFRunLoopCommonModes);
}

// 开始回调
void myRunLoopBeginCallback(CFRunLoopObserverRef observer, CFRunLoopActivity activity, void *info) {
    g_runLoopActivity = activity;
    g_runLoopMode = eRunloopDefaultMode;
    switch (activity) {
        // 进入 runloop
        case kCFRunLoopEntry:
            g_bRun = YES;
            break;
				// 处理 timers
        case kCFRunLoopBeforeTimers:
            if (g_bRun == NO) {
                gettimeofday(&g_tvRun, NULL);
            }
            g_bRun = YES;
            break;
				// 处理 sources
        case kCFRunLoopBeforeSources:
            if (g_bRun == NO) {
                gettimeofday(&g_tvRun, NULL);
            }
            g_bRun = YES;
            break;
				// 被唤醒
        case kCFRunLoopAfterWaiting:
            if (g_bRun == NO) {
                gettimeofday(&g_tvRun, NULL);
            }
            g_bRun = YES;
            break;
        case kCFRunLoopAllActivities:
            break;
        default:
            break;
    }
}

// 结束回调
void myRunLoopEndCallback(CFRunLoopObserverRef observer, CFRunLoopActivity activity, void *info) {
    g_runLoopActivity = activity;
    g_runLoopMode = eRunloopDefaultMode;
    switch (activity) {
        // 将要休眠
        case kCFRunLoopBeforeWaiting:
            gettimeofday(&g_tvRun, NULL);
            g_bRun = NO;
            break;
				// runloop 退出
        case kCFRunLoopExit:
            g_bRun = NO;
            break;

        case kCFRunLoopAllActivities:
            break;

        default:
            break;
    }
}
```

看这段代码的时候可能有点疑惑，为什么添加了两个 Observer。其实这两个 Observer 的 order 是不同的，为了保证在所有回调之前 beginCallback，在所有回调之后 endCallback，类似于 AutoreleasePool 的处理方式，具体可以看 [Matrix Issue 167](https://github.com/Tencent/matrix/issues/167) 下面的回答。

在监听的子线程我们只需要定期检测是否超过定义的卡顿时间就可以了，具体的代码如下

```objc
- (EDumpType)check
{
    // 是否在执行任务
    BOOL tmp_g_bRun = g_bRun;

    // runloop 上一次开始执行任务的时间
    struct timeval tmp_g_tvRun = g_tvRun;

    // check 时的时间
    struct timeval tvCur;
    gettimeofday(&tvCur, NULL);

    // 获取执行任务时间
    unsigned long long diff = [WCBlockMonitorMgr diffTime:&tmp_g_tvRun endTime:&tvCur];

    // 比较时间判断是否卡顿
}
```

#### 五、FOOM 监控

> FOOM（Foreground Out Of Memory），是指 App 在前台因消耗内存过多引起系统强杀。对用户而言，表现跟 crash 一样。(引用自 [iOS 微信内存监控](https://mp.weixin.qq.com/s/r0Q7um7P1p2gIb0aHldyNw))

关于 FOOM 的检测，Facebook 和微信的实现方式都是排除各种情况后剩余的情况就是 FOOM。

##### 5.1 程序的退出方式

- 正常退出
  - main 函数执行完返回
  - 调用 exit
  - 调用 \_exit
- 异常退出
  - 调用 abort
  - 进程收到某个信号终止

注：`assert`会先在控制台输出错误消息，然后调用`abort` 终止程序。exit 和 \_exit 的主要区别是 exit 会先调用 atexit() 回调、清理 IO 缓冲区，然后才调用 \_exit 退出程序。

##### 5.2 Matrix 实现

在程序的退出方式中，我们可以使用 `atexit` 函数注册终止处理程序来标记程序是通过 `exit` 退出的。同样的，我们可以在程序的生命周期通知和 Crash 之前标记程序退出，过滤退出条件判断是否 FOOM。Matrix 中定义的枚举如下

```objc
typedef enum : NSUInteger {
    MatrixAppRebootTypeBegin,
    MatrixAppRebootTypeOSVersionChange,  // 系统升级
    MatrixAppRebootTypeAPPVersionChange, // APP 升级
    MatrixAppRebootTypeOSReboot, 	     // 系统重启
    MatrixAppRebootTypeNormalCrash,      // Crash 退出
    MatrixAppRebootTypeQuitByExit,       // exit() 退出
    MatrixAppRebootTypeQuitByUser,       // 用户杀掉 APP
    MatrixAppRebootTypeAppSuspendOOM,    // OOM, out of memory
    MatrixAppRebootTypeAppSuspendCrash,  // WatchDog 干掉
    MatrixAppRebootTypeAppBackgroundOOM, // BOOM
    MatrixAppRebootTypeAppForegroundOOM, // FOOM
    MatrixAppRebootTypeAppForegroundDeadLoop,  // 死循环
    MatrixAppRebootTypeOtherReason,      // 其他
} MatrixAppRebootType;
```

##### 5.2.1 exit 注册终止函数标记

注册终止处理程序标记为 `MatrixAppRebootTypeQuitByExit`

```objc
+ (void)installAppRebootAnalyzer {
    atexit(g_matrix_app_exit);
}

void g_matrix_app_exit() {
    [MatrixAppRebootAnalyzer notifyAppQuitByExit];
}
```

##### 5.2.2 watchdog 标记

iOS 11 以上在启动卡顿的时候有很大概率被看门狗杀掉，一般时间是 20s，所以注册一个 `block` 在 18s 的时候写入标记。如果没有卡顿，在 APP 进入前台的时候执行 `dispatch_block_cancel` 掉 block。

```objc
NSTimeInterval now = [[NSDate date] timeIntervalSince1970];
s_suspendDelayBlock = dispatch_block_create(DISPATCH_BLOCK_NO_QOS_CLASS, ^{
    if ([[NSDate date] timeIntervalSince1970] - now < 20) {
        s_isSuspendKilled = YES;
        info.isAppSuspendKilled = YES;
        [info saveInfo];
        MatrixInfo(@"maybe killed after several seconds...");
    }
});
dispatch_after(dispatch_time(DISPATCH_TIME_NOW, (int64_t)(18 * NSEC_PER_SEC)), dispatch_get_main_queue(), s_suspendDelayBlock);
```

总结：FOOM 的判断比较依赖于 APP 退出场景的判断，Matrix 在 APP 的各种退出场景时写入标记，在下一次进入的时候进行判断是否是 FOOM。

#### 六、引用

感谢下列文章和书籍的作者大佬

- 深入解析 MacOS 和 iOS 操作系统
- 深入理解计算机系统
- [XNU-维基百科](https://zh.wikipedia.org/wiki/XNU)
- [BSD-维基百科](https://zh.wikipedia.org/wiki/BSD)
- [Darwin-维基百科](<https://zh.wikipedia.org/wiki/Darwin_(%E6%93%8D%E4%BD%9C%E7%B3%BB%E7%BB%9F)>)
- [Aqua(GUI)-维基百科](<https://zh.wikipedia.org/wiki/Aqua_(GUI)>)
- [解密 Runloop-MrPeak 杂货铺](http://mrpeak.cn/blog/ios-runloop/)
- [Cocoa Touch 框架](https://www.jianshu.com/p/a26fcbb3281a)
- [iOS Mach 异常和 signal 信号](https://yq.aliyun.com/articles/499180)
- [what-does-asm-volatile-do-in-c](https://stackoverflow.com/questions/26456510/what-does-asm-volatile-do-in-c)
- [iOS 微信内存监控](https://mp.weixin.qq.com/s/r0Q7um7P1p2gIb0aHldyNw)
- [ios/reducing-fooms-in-the-facebook-ios-app](https://code.fb.com/ios/reducing-fooms-in-the-facebook-ios-app/)
