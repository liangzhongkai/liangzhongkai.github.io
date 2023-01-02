---
title: linux时间系统
description: time
slug: linux_time
date: 2023-1-2 00:00:00+0000
categories:
    - linux
tags:
    - time
    - linux
disable_share: false
---

# 如何定义秒
在定义秒这个术语的时候是有两个标准的

1. 以铯133的振荡频率来定义秒
2. 依据地球自转和公转来定义秒

对于linux kernel而言，它采用了第一种定义的秒，并且参考点（linux epoch）也是UTC时间，UTC这个时间标准就是采用第一种方式来定义时间的。

POSIX标准曾经将时间参考点设定为1970年1月1日0点0分0秒（GMT），GMT就是采用第二种方法定义时间的，后面修正为UTC时间。

# linux时钟

硬件时钟：RTC(real time clock)

hardware clock, real time clock, RTC, BIOS clock以及CMOS clock等，在目前主流的服务器都采用RTC芯片实现，PC主板的芯片+电源供电。独立于OS，是最底层的时钟数据
RTC可以得到date and time. 可以在设备文件/dev/rtc进行RTC编程。也可以通过/sbin/clock配置时钟

软件时钟：OS时钟(os clock)

也是我们平时说的系统时间。PC主板的一个定时/计数芯片(OS时钟的基本单位就是该芯片的计数周期) -> 脉冲信号 -> 中断控制器 -> 中断信号 -> CPU call 时间中断服务程序维持OS时钟的正常工作

[RTC是OS时钟的时间基准，操作系统通过读取RTC来初始化OS时钟，此后二者保持同步运行<每隔一个固定时间会刷新或校正RTC中的信息>](https://blog.csdn.net/m0_37806112/article/details/80397338)


。。。。。。

linux, C++ 提供的时间函数都有一个致命的问题，它是在一个变频的cpu上衍生出来的。

有一个更加高的精度：rdtsc。它获取的是指令级别的精度。
