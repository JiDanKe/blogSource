title: 基于不同I/O策略的服务器模型
date: 2015-04-22 16:44:17
tags: [select, poll, epoll, I/O复用, 阻塞, 非阻塞, C10K, 服务器模型]
categories: 服务端基础
---
### 摘要

大多数时候我们编写的（Web）服务端程序，都是基于已有的网络服务框架或者服务器，这些服务端程序的关注点主要是处理领域业务，很少直接涉及到如何高效稳定处理与客户端的通信问题。所以，当你需要从服务端打得火热的各种框架中选择一个来开发应用程序，当你需要修改现有框架或优化现有应用，当你需要构建一个非Web网络服务程序，甚至当你需要从0到1编写一个网络服务器时，这些知识和经验还远远不够。

对于网络服务器程序来说，如何处理客户端连接是其前提和基础，能够稳定地处理大量的并发连接是服务器程序最重要的性能指标。由于操作系统任务处理和I/O模型的限制，我们可用的网络服务器模型是有限的。不同的服务模型可以应对不同的环境，具有不同的优点和缺点。

从操作系统任务处理的角度来看，有三种服务器模型可选：
* 多进程模型
* 多线程模型
* （单线程）异步模型

以上模型的特点及其应用，可以参考一些Web服务器的设计，这里不再赘述。后续会有相关的笔记进行稍微深入的讨论。

使用多进程或者多线程，其差异主要在于进程和线程的特性，所从I/O模型角度描述时我们使用线程来描述。从I/O模型的角度来看，最常用的网络服务器模型有：
* 单线程阻塞I/O模型
* 多线程阻塞I/O模型
* 单线程非阻塞I/O模型（包括非阻塞I/O,I/O复用）
* 单线程异步I/O模型

<!-- more -->

处理大量连接的时候，基于不同I/O策略的模型具有不同的资源（尤其是RAM和CPU）消耗和限制。

1. 单线程阻塞I/O模型：最简单的模型，计算机网络发展早期使用的解决方案。也是我们学习网络编程时首先接触的例子。它使用循环的方式，遍历处理所有的连接。这种方式的问题显而易见，随着连接数的增加，低效又缓慢。

2. 多线程阻塞I/O模型：这是利用多线程对 **单线程阻塞I/O模型** 的改进，每个线程负责监听处理一个连接。随之而来的问题是，开启销毁线程的高昂代价：线程栈对内存的需求，线程切换对CPU的需求，频繁线程切换造成的不稳定（线程抖动）。

3. 单线程非阻塞I/O模型：与 **单线程阻塞I/O模型** 类似，循环遍历处理所有连接。只是由于非阻塞I/O的特点，内核没有可用数据时立即返回，不会阻塞线程。这样的循环轮询容易大幅度推高CPU的占用率，因此一般我们不会使用这种方式，而是引入I/O复用，当没有可用数据时将线程阻塞在代理（select/poll/epoll）调用上，占用较少的CPU资源。

4. 单线程异步I/O模型：基于异步I/O的模型，我没有研究过相应的框架，不熟悉。

上述模式是构建服务器的基础，在实践中我们还会在此基础上会使用线程池、进程池等技术进一步优化程序。根据应用的特点，结合多种技术的优点以获得相对最优的选择。例如，***在单线程非阻塞I/O模型下，对于I/O密集型应用和计算（CPU）密集型应用，可以有不同的处理方式。对于I/O密集型应用，直接在循环监听线程中处理即可（Node.js/Tornado）；对于计算（CPU）密集型应用，则（选择）使用多线程/进程来处理就绪的连接，避免阻塞线程对连接的监听（术语叫event based，Nginx/Apache Event模式）***。

以上关于服务器模型的选择，先入为主，大而化之地说了结论。对在实践中如何选择，应该注意什么问题没有做深入的说明。Dan Kegel在[《The C10K problem》](http://www.kegel.com/c10k.html)的 “I/O Strategies” 章节中做了详细而深入的论述，这篇比价的后续内容就是对这节的摘要，更详细的内容请参考原文（虽然现在面临的不止C10K，但其理论依然很重要）。

### I/O 策略

基于不同I/O策略，实现网络服务器时一般需要考虑下列问题：

1. 是否需要在单个线程中处理多个I/O调用？

    * 不处理，始终使用同步阻塞I/O（blocking/synchronous）调用，使用多进程和多线程来实现并发支持；

    * 使用非阻塞I/O调用读取I/O，当I/O完成时发出通知（如selelct，poll，/dev/poll）从而开始下一个I/O。这种I/O主要用于网络I/O上而不是磁盘I/O上；

    * 使用异步I/O，当I/O完成时会发出通知（AIO/IOCP，异步I/O或完成端口），可以同时使用在网络和磁盘I/O上。


2. 如何控制对每个客户端的服务？

	* 对每一个客户端使用一个进程（经典的UNIX方法，从1980年一直使用）；
	* 一个系统级线程处理多个客户端，每个客户端是如下一种：
		* a user-level thread (e.g. GNU state threads, classic Java with green threads)：用户级线程（比如GNU状态线程，JAVA绿色线程【一种虚拟机级别的线程，和本地操作系统线程不是一一对应的关系】）

		* a state machine (a bit esoteric, but popular in some circles; my favorite)：状态机（有点复杂）

		* a continuation (a bit esoteric, but popular in some circles)

	* 一个系统级线程对应一个客户端（比如JAVA中的本地线程）

	* 一个系统级的线程对应每一个活动的客户端(比如 Tomcat with apache front end; NT完成端口; 线程池)

3. 是使用标准的操作系统服务还是把一些代码放入内核（比如自定义驱动，内核模块，VxD）

综合对上述问题的考虑，平常主要用到下面5中服务器模型：


1. 一个线程服务多个客户端，使用非阻塞I/O和水平触发的就绪通知
2. 一个线程服务多个客户端，使用非阻塞I/O和就绪改变时通知（或者边缘触发的就绪通知）
3. 一个线程服务多个客户端，使用异步I/O
4. 一个线程服务一个客户端，使用阻塞I/O
5. 把服务代码编译进内核


#### 一个线程服务多个客户端，使用非阻塞I/O和水平触发的就绪通知

传统模型，使用select/poll，在此模型下由内核告知某个描述符已经准备就绪（“水平触发”模式，只要不处理事件，就会一直有通知。相对的是“边缘触发”，在Jonathon Lemon [关于kqueue()的论文](http://people.freebsd.org/~jlemon/papers/kqueue.pdf)中介绍了这两个术语）。

这种模型在磁盘I/O操作时会有一个问题，当页不在内存中或者内存映射磁盘文件时，设置不阻塞标识是没有效果的。

在非阻塞套接字的集合中，关于单一线程是如何告知哪个套接字准备就绪的，以下列出了几种方法：

* 传统的select()
遗憾的是，select()受限于FD_SETSIZE个句柄。该限制被编译进了标准库和用户程序（有些版本的C library允许你在用户程序编译时放宽该限制）。


* 传统的poll()
poll()虽然没有文件描述符个数的硬编码限制，但是当有数千个时速度就会变得很慢，因为 大多数的文件描述符在某个时间是空闲的，彻底扫描数千个描述符是需要花费一定时间的。
有些操作系统（如Solaris 8）通过使用了poll hinting技术改进了poll()，该技术由Niels Provos在1999年实现并利用基准测试程序测试过。

* /dev/poll
	这是在Solaris中被推荐的代替poll的方法。

    /dev/poll的背后思想就是利用poll()在大部分的调用时使用相同的参数。使用/dev/poll时 ，首先打开/dev/poll得到文件描述符，然后把你关心的文件描述符写入到/dev/poll的描述符， 然后你就可以从/dev/poll的描述符中读取到已就绪的文件描述符。

    /dev/poll 在Solaris 7(see patchid 106541) 中就已经存在，不过在Solaris 8 中才公开现身。在750个客户端的情况下，this has 10% of the overhead of poll()。
    关于/dev/poll在Linux上有多种不同的尝试实现，但是没有一种可以和epoll相比，不推荐在 Linux上使用/dev/poll。

* kqueue()
这是在FreeBSD系统上推荐使用的代替poll的方法(and, soon, NetBSD)。
***kqueue()即可以水平触发，也可以边缘触发。***


#### 一个线程服务多个客户端，使用非阻塞I/O和就绪改变时通知（或者边缘触发的就绪通知）

Readiness change notification（或边缘触发就绪通知）的意思就是当你给内核一个文件描述符，一段时间后，如果该文件描述符从没有就绪到已经准备就绪，那么内核就会发出通知，告知该文件描述符已经就绪，***并且不会再对该描述符发出类似的就绪通知直到你在描述符上进行一些操作使得该描述符不再就绪***（如直到在send，recv或者accept等调用上遇到EWOULDBLOCK错误，或者发送/接收了少于需要的字节数）。

由以下一些方法来支持，描述符已经准备就绪通知：

* kqueue()
FreeBSD 4.3及以后版本，NetBSD（2002.10）都支持 kqueue()/kevent()， 支持边沿触发和水平触发（请查看Jonathan Lemon 的网页和他的BSDCon 2000关于kqueue的论文）。

 就像/dev/poll一样，你分配一个监听对象，不过不是打开文件/dev/poll，而是调用kqueue ()来获得。需要改变你所监听的事件或者获得当前事件的列表，可以在kqueue()返回的描述符上 调用kevent()来达到目的。它不仅可以监听套接字，还可以监听普通的文件的就绪，信号和I/O完 成的事件也可以。


* epoll()
这是Linux 2.6的内核中推荐使用的边沿触发poll。


* Realtime Signals实时信号（略，具体请参考资料）
* Signal-per-fd（略，具体请参看资料）



#### 一个线程服务多个客户端，使用异步I/O

该方法目前还没有在Unix上普遍的使用，可能因为很少的操作系统支持异步I/O，或者因为它需要重新修改应用程序(rethinking your applications)。 在标准Unix下，异步I/O是由“aio_”接口 提供的，它把一个信号和值与每一个I/O操作关联起来。信号和其值的队列被有效地分配到用户的 进程上。异步I/O是POSIX 1003.1b实时标准的扩展，也属于Single Unix Specification,version 2.

AIO使用的是边缘触发的完成时通知，例如，当一个操作完成时信号就被加入队列（也可以使用“水平触发”的完成时通知，通过调用aio_suspend()即可， 不过我想很少人会这么做）。

***注意AIO并没有提供无阻塞的为磁盘I/O打开文件的方法，如果你在意因打开磁盘文件而引起 sleep的话，Linus建议 你在另外一个线程中调用open()而不是把希望寄托在对aio_open()系统调用上。***

在Windows下，异步I/O与术语“重叠I/O”和“IOCP”(I/O Completion Port,I/O完成端口)有一定联系。Microsoft的IOCP结合了先前的如异步I/O(如aio_write)的技术，把事件完成的通知进行排队(就像使用了aio_sigevent字段的aio_write),并且它为了保持单一IOCP线程的数量从而阻止了一部分请求。

#### 一个线程服务一个客户端，使用阻塞I/O

让read()和write()阻塞. 这样不好的地方在于需要为每个客户端使用一个完整的线程栈，从而比较浪费内存。 许多操作系统仍在处理数百个线程时存在一定的问题。如果每个线程使用2MB的栈，那么当你在32位的机器上运行 512（2^30 / 2^21=512）个线程时，你就会用光所有的1GB的用户可访问虚拟内存（Linux也是一样运行在x86上的）。你可以减小每个线程所拥有的栈内存大小，但是由于大部分线程库在一旦线程创建后就不能增大线程栈大小，所以这样做就意味着你必须使你的程序最小程度地使用内存。当然你也可以把你的程序运行在64位的处理器上去。

Linux，FreeBSD和Solaris系统的线程库一直在更新，64位的处理器也已经开始在大部分的用户中所使用。 也许在不远的将来，这些喜欢使用一个线程来服务一个客户端的人也有能力服务于10000个客户了。 但是在目前，如果你想支持更多的客户，你最好还是使用其它的方法。

**Note: 1:1 threading vs. M:N threading**

**在实现线程库的时候有一个选择就是你可以把所有的线程支持都放到内核中（也就是所谓的1：1的模型），也可以 把一些线程移到用户空间上去（也就是所谓的M：N模型）。从某个角度来说, M:N被认为拥有更好的性能，但是由于很难被正确的编写， 所以大部分人都远离了该方法。**

* [为什么Ingo Molnar相对于M：N更喜欢1：1](http://marc.theaimsgroup.com/?l=linux-kernel&m=103284879216107&%20w=2)
* [Sun改为1：1的模型](http://java.sun.com/docs/hotspot/threads/threads.html)
* [NGPT](http://www-124.ibm.com/pthreads/)是Linux下的M：N线程库.
* Although [Ulrich Drepper计划在新的glibc线程库中使用M：N的模型](http://people.redhat.com/drepper/glibcthreads.html), 但是还是[选用了1：1的模型](http://people.redhat.com/drepper/nptl-design.pdf).
* [MacOSX 也将使用1：1的线程.](http://developer.apple.com/technotes/tn/tn2028.html#MacOSXThreading)
* [FreeBSD](http://people.freebsd.org/~julian/) 和 [NetBSD](http://web.mit.edu/nathanw/www/usenix/freenix-sa/) 仍然将使用M：N线程，FreeBSD 7.0也倾向于使用1：1的线程（见上面描述），可能M：N线程的拥护者最后证明它是错误的。


#### 把服务代码编译进内核

Novell和Microsoft都宣称已经在不同时期完成了该工作，至少NFS的实现完成了该工作。 khttpd在Linux下为静态web页面完成了该工作， Ingo Molnar完成了"TUX" (Threaded linUX webserver) ，这是一个Linux下的快速的可扩展的内核空间的HTTP服务器。 Ingo在2000.9.1宣布 alpha版本的TUX可以在 ftp://ftp.redhat.com/pub/redhat/tux下载, 并且介绍了如何加入其邮件列表来获取更多信息。

在Linux内核的邮件列表上讨论了该方法的好处和缺点，多数人认为不应该把web服务器放进内核中，相反内核加入最小的钩子hooks来提高web服务器的性能，这样对其它形式的服务器就有益。 具体请看 Zach Brown的讨论 对比用户级别和内核的http服务器。在2.4的linux内核中为用户程序提供了足够的权限（power），就像X15 服务器运行的速度和TUX几乎一样，但是它没有对内核做任何改变。

### 参考资料

1. Dan Kegel [《The C10K problem》](http://www.kegel.com/c10k.html) 原文， 中文译文参考 [http://www.cnblogs.com/fll/archive/2008/05/17/1201540.html](http://www.cnblogs.com/fll/archive/2008/05/17/1201540.html)。

2. 顾 锋磊, [使用事件驱动模型实现高效稳定的网络服务器程序](http://www.ibm.com/developerworks/cn/linux/l-cn-edntwk/)

3. Martin C. Brown, [使用libevent和libev提高程序的网络性能](http://www.ibm.com/developerworks/cn/aix/library/au-libev/)
