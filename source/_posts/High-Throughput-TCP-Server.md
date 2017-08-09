---
title: High Throughput TCP Server
date: 2014-04-09 20:56:28
tags:
  - TCP
  - High Throughput
---

在Linux上开发高吞吐量、高并发，即C100K的Server时，默认的一些系统参数已经不能满足性能的需求，从而会导致性能瓶颈。现记录一些自己在学习过程中已知的解决方案。（C1000K的解决方案与此不同，要从内核入手。）

1、打开文件数限制
----------------

使用命令`ulimit -a`可以查看系统给出的所有限制。其中，由`-n`指出的open files限制了一个进程同时能够打开的最大的文件数，其实也就是最大的文件描述符数。由于在程序中每一个打开的TCP连接也都由系统抽象成一个文件，因此这个值也直接限制了进程可以打开的TCP连接的数量。

修改办法：在/etc/security/limits.conf中添加：

```bash
*                soft    nofile          793937
*                hard    nofile          793937
```

其中，793937指定最大可以打开文件数量，这个值由系统通过内存大小来推荐，通过`cat /proc/sys/fs/file-max`或`sysctl fs.file-max`来查看。如果系统默认的fs.file-max值也过小，那么也可以直接修改这个值。

<!-- more -->

2、内核参数
----------

使用命令`sysctl -a`可以查看所有的内核参数，这些参数也可以通过Linux上的虚拟文件系统/proc来查看。对Server呑吐量有直接影响的主要参数有：

`net.core.somaxconn`限制着在一个socket上排队的连接的总数，即`listen(int sockfd, int backlog)`函数中backlog参数的最大值。在一个socket上排队的连接有两种：已经成功建立的和尚未成功建立的。当TCP服务端监听某个地址时，TCP客户端通过向该地址发送一个SYN分节来发起连接。当服务器依次经历：1.接收到这个SYN包， 2.返回一个ACK、SYN分节， 3.接收到来自客户端的ACK分节，之后连接便正式建立。步骤3之后的连接，在Server调用accept函数时能成功返回给应用程序，是属于已经成功建立的连接。而步骤1到3之间的连接，则是还未成功建立的连接，accept函数不会返回这样的连接。由于要支持大呑吐量的Server，这个参数需要设置得大一点，比如1024、4096等。

`net.ipv4.tcp_max_syn_backlog`限制着socket发出SYN分节后等待ACK回应的连接的个数。当压力大时，该值也尽量设置大一些。

`fs.epoll.max_user_watches`限制着每个用户可以在整个系统的epoll实例上注册的描述符的数量。在现在的机器上，这个数量一般都很大，但是如果要做比如长连接的server的时候，这个参数就需要考虑了。

另外，还有一些参数用来调节Timeout时间、内核缓冲区大小等等，可以根据具体的经验来设置。

3、网卡硬中断/软中断平衡
-----------------------

**硬中断**：多队列网卡支持将IO中断分配到多个CPU核心上。通过命令`lspci -vvv`可以查看所有PCI设备的详细信息，找到网卡的IRQ号。然后`cat /proc/interrupts`可以查看各中断线在各CPU核心上的分布。使用`watch -n 1 "cat /proc/interrupts"`可以对其进行实时监控。如果发现网卡中断几乎全被CPU0包下来了，那么可以调整一下，让多CPU共同承担中断处理。比如：

```bash
echo f > /proc/irq/{中断号}/smp_affinity
```

用来将硬中断交给0-3号CPU。（这个设置与irqbalance服务可能会冲突，还没研究…… 另外对于单队列的网卡，以上设置是无效的。）

**软中断**：由于驱动程序会尽快结束上半部的工作，而把主要的事情放到下半部去做，因此软中断对CPU的消耗是相当巨大的。RPS/RFS通过把软中断平衡到多个CPU上能得到性能的极大提升。通过

```bash
echo f | sudo tee /sys/class/net/eth0/queues/rx-$i/rps_cpus （i取值目录下的全部）
```

可以将软中断分散到各CPU核心上。之后，可以通过`mpstat -P ALL 1`可以监控CPU各核心的softirq。（使用tee是因为普通文件重定向是由bash完成，即使使用sudo echo，bash本身也是没有重定向到该文件的权限的。）

4、使用合理的IO模型
------------------

文章C10K Problem里讲述了各种IO模型。一般来讲，在高并发的情况下，使用epoll或kqueue做就绪通知，实现非阻塞的IO是效率较高的。例如Python的Tornado、Java的Netty/NIO等。并且，在多核的机器上还可以使用多进程/线程来处理客户端的连接请求和数据IO，以进一步提高性能。等等。

