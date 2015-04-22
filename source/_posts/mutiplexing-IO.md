title: I/O 复用：select、poll和epoll
date: 2015-04-22 15:35:16
tags: [select, poll, epoll, I/O mutiplexing, blocking, nonblocking]
categories: 服务端基础
---
#### 引言

在**阻塞式I/**O模型下，一个线程只能处理一个流的I/O事件，为了同时高效处理多个流，只能采用多进程（fork）或者多线程的方式。

使用**非阻塞I/O**模型，可以通过循环遍历的方式同时处理多个流（忙轮询），但是如果所有流都没有数据，就会造成CPU的浪费（在一些服务端应用程序上便不高效）。

为了避免CPU的浪费，引入一个代理（**select/poll，I/O复用**），这个代理能够同时监视多个流的事件，当没有流事件发生时阻塞调用线程，释放CPU资源。

虽然select/poll能够同时监视多个流（描述符）事件，但是并不知道具体是哪一个流的事件发生，其实现是遍历所有的描述符，故称为无差别轮询。随着监视的描述符增多，性能会下降，并且有些系统实现里面单个进程监视的描述符数量是有最大限制的。

所以有了epoll（linux）/kqueue（freebsd）/iocp（windows）的出现，不同于非阻塞式I/O的忙轮询和I/O复用的无差别轮询，epoll只在某个流事件发生时通知调用者（内核实现）。

#### select

调用select告诉内核对那些描述符（就读、写或异常条件）感兴趣以及等待多长时间。感兴趣的描述符不局限于套接字，任何描述符都可以使用select来测试。以下为select的函数签名：

{% asset_img select.png %}

select 函数监视的文件描述符分3类，分别是readfds、writefds和exceptfds（都是值-结果参数）。调用后select函数会阻塞，直到有描述符就绪（有数据可读、可写、或者有异常）或者超时（timeout指定等待时间， ~~如果立即返回设为null即可~~），函数返回。当select函数返回后，可以 通过遍历fdset，来找到就绪的描述符。

timeout告知内核等待所指定的描述符中的任何一个就绪可花多长时间。其timeval结构用于指定这段时间的秒数和微妙数（虽然允许指定，实际上由于操作系统内核实现的原因这个微秒级的分辨率实际上并不很精确）。
<!-- more -->
{% asset_img timeval.png %}

这个参数有3种可能：
1. 永远等待下去：仅在有一个描述符准备好I/O时才返回。为此，参数需要设置为null；
2. 等待一段时间：在有一个描述符准备好I/O时返回，但是不超过由该参数指定的时间；
3. 根本不等待，立即返回，成为轮询(polling)。为此，该参数必须指向一个timeval结构，而且其中的定时器值（tv_sec,tv_usec）必须设置为0。

select目前几乎在所有的平台上支持，其良好跨平台支持也是它的一个优点。select的一 个缺点在于单个进程能够监视的文件描述符的数量存在最大限制，在Linux上一般为1024（select函数中参数maxfdp1参数指定了等待测试的描述符的个数。头文件<**sys/select.h**>中定义了**FD_SETSIZE**来限定最大值，一般为1024），可以通过~~修改宏定义~~（*注：这个我记得好像无效，不确定*）甚至重新编译内核的方式提升这一限制，但是这样也会造成效率的降低。

#### poll

{% asset_img poll.png %}

不同于select使用（中间）三个fd_set值-结果参数的方式，poll使用一个 pollfd结构指针。

{% asset_img pollfd.png %}

pollfd结构包含了要监视的event和发生的event，不再使用select“值-结果”的方式。每个描述符有两个变量，一个为调用值（指定需要监视的事件），一个返回结果（返回发生的事件）。同时，pollfd并没有最大数量限制（但是数量过大后性能也是会下降）。

nfds参数指定结构数组的元素个数。

timeout参数指定poll函数返回前等待多长时间，可能值为
1. *INFTIM ，永远等待*；
2. *0，立即返回不阻塞进程；*
3. *\>0，等待指定数据的毫秒数。*

和select函数一样，poll返回后，需要轮询pollfd来获取就绪的描述符。

***select和poll都需要在返回后，通过遍历文件描述符来获取已经就绪的socket。事实上，同时连接的大量客户端在一时刻可能只有很少的处于就绪状态，因此随着监视的描述符数量的增长，其效率也会线性下降。***

#### epoll(Linux 2.6)
epoll的接口如下：
```c
epoll_create( size)；
epoll_ctl( epfd,  op,  fd, struct epoll_event *event)；
            typedef union epoll_data {
                * fd;
                __uint32_t u32;
                __uint64_t u64;
            } epoll_data_t;

            struct epoll_event {
                __uint32_t events;      /* Epoll events */
                epoll_data_t data;     /*  User data variable */
         }
  epoll_wait( epfd, struct epoll_event * events,  maxevents,  timeout);
```

主要是epoll_create，epoll_ctl和epoll_wait三个函数。
1. epoll_create函数创建epoll文件描述符，参数size并不是限制了epoll所能监听的描述符最大个数，只是对内核初始分配内部数据结构的一个建议。返回是epoll描述符。-1表示创建失败。

2. epoll_ctl 控制对指定描述符fd执行op操作，event是与fd关联的监听事件。op操作有三种：添加EPOLL_CTL_ADD，删除EPOLL_CTL_DEL，修改EPOLL_CTL_MOD。分别添加、删除和修改对fd的监听事件。event可以是以下几个宏的集合：
    * **EPOLLIN** ：表示对应的文件描述符可以读（包括对端SOCKET正常关闭）；
    * **EPOLLOUT**：表示对应的文件描述符可以写；
    * **EPOLLPRI**：表示对应的文件描述符有紧急的数据可读（这里应该表示有带外数据到来）；
    * **EPOLLERR**：表示对应的文件描述符发生错误；
    * **EPOLLHUP**：表示对应的文件描述符被挂断；
    * **EPOLLET**： 将EPOLL设为边缘触发(Edge Triggered)模式，这是相对于水平触发(Level Triggered)来说的。
    * **EPOLLONESHOT**：只监听一次事件，当监听完这次事件之后，如果还需要继续监听这个socket的话，需要再次把这个socket加入到EPOLL队列里。

3. epoll_wait 等待epfd上的io事件，最多返回maxevents个事件。
在 select/poll中，进程只有在调用一定的方法后，内核才对所有监视的文件描述符进行扫描，而epoll事先通过epoll_ctl()来注册一 个文件描述符，一旦基于某个文件描述符就绪时，内核会采用类似callback的回调机制，迅速激活这个文件描述符，当进程调用epoll_wait() 时便得到通知。

epoll的优点主要是一下几个方面：
1. 监视的描述符数量不受限制，它所支持的FD上限是最大可以打开文件的数目，这个数字一般远大于2048,举个例子,在1GB内存的机器上大约是10万左 右，具体数目可以 `cat /proc/sys/fs/file-max` 查看,一般来说这个数目和系统内存关系很大。select的最大缺点就是进程打开的fd是有数量限制的。这对于连接数量比较大的服务器来说根本不能满足，在Linux上使用多进程是常见的可行解决方案( Apache就是这样实现的)。虽然linux上面创建进程的代价比较小，但仍旧是不可忽视的，加上进程间数据同步远比不上线程间同步的高效，所以也不是一种完美的方案。

2. IO的效率不会随着监视fd的数量的增长而下降。epoll不同于select和poll轮询的方式，而是通过为每个fd定义回调函数来支持的，也就是说，只有就绪的fd才会执行回调函数。

3. 支持水平触发和边沿触发（*只告诉进程哪些文件描述符刚刚变为就绪状态，它只说一遍，如果我们没有采取行动，那么它将不会再次告知，这种方式也称为边缘触发*）两种方式，理论上边沿触发的性能要更高一些，但是代码实现相当复杂。

4. mmap加速内核与用户空间的信息传递。epoll是通过内核与用户空间mmap同一块内存，避免了无畏的内存拷贝。

#### 参考资料

1. UNIX I/O模型参考引用[《UNIX网络编程：卷一》](http://book.douban.com/subject/1500149/)第六章。
