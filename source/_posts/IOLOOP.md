title: tornado：IOLoop模块解析
date: 2015-04-22 21:04:01
tags: [python, tornado, ioloop, select, poll, epoll, kqueue]
categories: tornado
---
### 摘要

[* * * `ioloop.py` * * *](https://github.com/tornadoweb/tornado/blob/v4.0.2/tornado/ioloop.py#L17) 是整个tornado的核心模块，负责实现服务器的异步非阻塞机制。其中[** `IOLoop` **](https://github.com/tornadoweb/tornado/blob/v4.0.2/tornado/ioloop.py#L70)类是一个基于level-triggered的I/O事件循环，它使用I/O多路复用模型（select模型）监视每个文件描述符的I/O事件是否就绪，当文件描述符I/O事件就绪后调用对应的处理器（handler）进行处理。本篇笔记是对tornado v4.0.3的 IOLoop模块的源码解析。

### IOLoop

IOLoop在Linux下使用`epoll`, 在BSD/Mac OS X下使用`kqueue`，否则使用`selelct`。

```python
    @classmethod
    def configurable_default(cls):
        if hasattr(select, "epoll"):
            from tornado.platform.epoll import EPollIOLoop
            return EPollIOLoop
        if hasattr(select, "kqueue"):
            # Python 2.6+ on BSD or Mac
            from tornado.platform.kqueue import KQueueIOLoop
            return KQueueIOLoop
        from tornado.platform.select import SelectIOLoop
        return SelectIOLoop
```

通过调用[`add_handler`](https://github.com/tornadoweb/tornado/blob/v4.0.2/tornado/ioloop.py#L674)方法将一个文件描述符(v4.0中增加了对file-like object的支持)加入到I/O事件循环中：

```python
    def add_handler(self, fd, handler, events):
        """Registers the given handler to receive the given events for ``fd``.

        The ``fd`` argument may either be an integer file descriptor or
        a file-like object with a ``fileno()`` method (and optionally a
        ``close()`` method, which may be called when the `IOLoop` is shut
        down).

        The ``events`` argument is a bitwise or of the constants
        ``IOLoop.READ``, ``IOLoop.WRITE``, and ``IOLoop.ERROR``.

        When an event occurs, ``handler(fd, events)`` will be run.

        .. versionchanged:: 4.0
           Added the ability to pass file-like objects in addition to
           raw file descriptors.
        """

        fd, obj = self.split_fd(fd)
        self._handlers[fd] = (obj, stack_context.wrap(handler))
        self._impl.register(fd, events | self.ERROR)
```
<!-- more -->

`self.split_fd`方法将文件描述符或者file-like object包装成（文件描述符，object）,`self._handlers`字段保存文件描述符对应的处理器，然后将需要监视的I/O事件注册到select中。在Tornado中只关心`READ`, `WRITE`, 和 `ERROR`事件，其中`ERROR`事件是自动添加的。

`self._impl`是`select.epoll`、`select.select`、`select.kqueue`中的任一Tornado实现，其中
 1. `select` 对应 [`tornado.platform.select.SelectIOLoop，impl=_Select`](https://github.com/tornadoweb/tornado/blob/v4.0.2/tornado/platform/select.py#L74)

 2. `epoll` 对应 [`tornado.platform.epoll.EPollIO`](https://github.com/tornadoweb/tornado/blob/v4.0.2/tornado/platform/epoll.py#L24)
 
 3. `kqueue`对应 [`tornado.platform.kqueue.KQueueIOLoop，impl=_KQueue`](https://github.com/tornadoweb/tornado/blob/v4.0.2/tornado/platform/kqueue.py#L90)

`select.select`，`select.kqueue` 分别通过 `_Select`、`_KQueue` 接口适配到 `select.epoll`。


调用`IOLoop.start`方法启动I/O循环直到`IOLoop.stop`方法被调用才会停止（注意：stop方法只是设置停止标识，循环必须在处理完当前的I/O事件后才退出）。start方法封装了I/O循环的处理流程，其代码如下所示：
```python
    def start(self):

        # tonardo使用_running/_stopped两个字段组合表示3种状态：
        # 1、就绪（初始化完成/已经结束）：_running=False, _stopped=False;
        # 2、正在运行：_running=True, _stopped=False;
        # 3、正在结束：_running=False, _stopped=True;
        if self._running:
            raise RuntimeError("IOLoop is already running")
        self._setup_logging()
        if self._stopped:
            self._stopped = False
            return
        old_current = getattr(IOLoop._current, "instance", None)
        IOLoop._current.instance = self
        self._thread_ident = thread.get_ident()
        self._running = True

        # signal.set_wakeup_fd closes a race condition in event loops:
        # a signal may arrive at the beginning of select/poll/etc
        # before it goes into its interruptible sleep, so the signal
        # will be consumed without waking the select.  The solution is
        # for the (C, synchronous) signal handler to write to a pipe,
        # which will then be seen by select.
        #
        # In python's signal handling semantics, this only matters on the
        # main thread (fortunately, set_wakeup_fd only works on the main
        # thread and will raise a ValueError otherwise).
        #
        # If someone has already set a wakeup fd, we don't want to
        # disturb it.  This is an issue for twisted, which does its
        # SIGCHILD processing in response to its own wakeup fd being
        # written to.  As long as the wakeup fd is registered on the IOLoop,
        # the loop will still wake up and everything should work.
        #
        # signal.set_wakeup_fd(fd)设置文件描述符fd, 当接收到信号时会在它上面写入一个 '＼0' 字节。
        # 用于唤醒被poll或select调用阻塞的进程，使进程能够处理信号。方法参数fd必须是以非阻塞
        # (non-blocking)方式打开的文件描述符，否则无效。调用该方法返回上一次调用设置的文件描述符（没有
        # 设置过则返回-1）。该方法只能在主线程中调用，在其他线程调用时将抛出 ValueError 异常。
        #
        # 上述原注释中有提到twisted自身会设置wakeup fd处理SIGCHILD信号，所以在结合twisted使用时要注
        # 意override PosixReactorBase.installWaker等与waker相关法方法（暂时对twistd不了解，猜测）。
        #
        # self._waker.write_fileno()文件描述符的 READ 事件已经在 initialize 方法中加入I/O循环列表。
        old_wakeup_fd = None
        if hasattr(signal, 'set_wakeup_fd') and os.name == 'posix':
            # requires python 2.6+, unix.  set_wakeup_fd exists but crashes
            # the python process on windows.
            try:
                old_wakeup_fd = signal.set_wakeup_fd(self._waker.write_fileno())
                if old_wakeup_fd != -1:
                    # Already set, restore previous value.  This is a little racy,
                    # but there's no clean get_wakeup_fd and in real use the
                    # IOLoop is just started once at the beginning.
                    signal.set_wakeup_fd(old_wakeup_fd)
                    old_wakeup_fd = None
            except ValueError:  # non-main thread
                pass

        try:
            while True:
                # Prevent IO event starvation by delaying new callbacks
                # to the next iteration of the event loop.
                with self._callback_lock:
                    callbacks = self._callbacks
                    self._callbacks = []

                # Add any timeouts that have come due to the callback list.
                # Do not run anything until we have determined which ones
                # are ready, so timeouts that call add_timeout cannot
                # schedule anything in this iteration.
                #
                # self._timeouts是一个基于 heap 的 priority queue，存放_Timeout类型实例，
                # 按照到期时间由近到远和加入heap的先后顺序排序（参见_Timeout的__lt__和__le__）。
                due_timeouts = []
                if self._timeouts:
                    now = self.time()
                    while self._timeouts:
                        if self._timeouts[0].callback is None:
                            # The timeout was cancelled.  Note that the
                            # cancellation check is repeated below for timeouts
                            # that are cancelled by another timeout or callback.
                            heapq.heappop(self._timeouts)
                            self._cancellations -= 1
                        elif self._timeouts[0].deadline <= now:
                            due_timeouts.append(heapq.heappop(self._timeouts))
                        else:
                            break

                    # 由于从heap中移除一个元素很复杂，所以tronado实现remove_timeout时将取消的
                    # timeout对象保留在heap中，这样可能会导致内存问题，所以这里做了一个处理512的
                    # 阈值执行垃圾回收。remove_timeout方法的注释中有说明。
                    if (self._cancellations > 512
                            and self._cancellations > (len(self._timeouts) >> 1)):
                        # Clean up the timeout queue when it gets large and it's
                        # more than half cancellations.
                        self._cancellations = 0
                        self._timeouts = [x for x in self._timeouts
                                          if x.callback is not None]
                        heapq.heapify(self._timeouts)

                for callback in callbacks:
                    self._run_callback(callback)
                for timeout in due_timeouts:
                    if timeout.callback is not None:
                        self._run_callback(timeout.callback)
                # Closures may be holding on to a lot of memory, so allow
                # them to be freed before we go into our poll wait.
                #
                # 在进入poll等待之前释放闭包占用的内存，优化系统
                callbacks = callback = due_timeouts = timeout = None

                # 优化poll等待超时时间：
                # 1、I/O循环有callback需要处理时，不阻塞poll调用，也就是poll_timeout=0；
                # 2、I/O循环有timeout需要处理时，计算第一个timeout（self._timeouts[0]，
                #    最先超时需要处理的timeout）距离现在的超时间隔，取poll_timeout默认值与
                #    该间隔之间的最小值（以保证timeout 一超时就能被I/O循环立即处理，不被poll
                #    等待导致延时；若第一个timeout现在已经超时，则最小值<0，故需要与0比较修正）；
                # 3、I/O循环没有callback和timeout需要处理，则使用默认等待时间。
                if self._callbacks:
                    # If any callbacks or timeouts called add_callback,
                    # we don't want to wait in poll() before we run them.
                    poll_timeout = 0.0
                elif self._timeouts:
                    # If there are any timeouts, schedule the first one.
                    # Use self.time() instead of 'now' to account for time
                    # spent running callbacks.
                    poll_timeout = self._timeouts[0].deadline - self.time()
                    poll_timeout = max(0, min(poll_timeout, _POLL_TIMEOUT))
                else:
                    # No timeouts and no callbacks, so use the default.
                    poll_timeout = _POLL_TIMEOUT

                if not self._running:
                    break

                # 为了监视I/O循环的阻塞状态，tornado提供了通过定时发送SIGALRM信号的方式来异步通知
                # 进程I/O循环阻塞超过了预期的最大时间（self._blocking_signal_threshold）。
                #
                # IOLoop.set_blocking_signal_threshold()方法设置一个signal.SIGALRM
                # 信号处理函数来监视I/O循环的阻塞时间。
                #
                # poll调用返回后（poll等待时间不计入I/O循环阻塞时间），通过调用signal.setitimer(
                # signal.ITIMER_REAL, self._blocking_signal_threshold, 0)设置定时器，每间
                # 隔 _blocking_signal_threshold 发送一个 SIGALRM 信号，也就是说当I/O循环阻塞超
                # 过 _blocking_signal_threshold 时会发送一个 SIGALRM 信号。
                #
                # 进入poll之前调用signal.setitimer(signal.ITIMER_REAL, 0, 0)清理定时器，直到
                # poll返回后重新设置定时器。
                if self._blocking_signal_threshold is not None:
                    # clear alarm so it doesn't fire while poll is waiting for
                    # events.
                    signal.setitimer(signal.ITIMER_REAL, 0, 0)

                try:
                    event_pairs = self._impl.poll(poll_timeout)
                except Exception as e:
                    # Depending on python version and IOLoop implementation,
                    # different exception types may be thrown and there are
                    # two ways EINTR might be signaled:
                    # * e.errno == errno.EINTR
                    # * e.args is like (errno.EINTR, 'Interrupted system call')
                    #
                    # poll调用可能会导致进程进入阻塞状态（sleep），这时候进程被某个系统信号唤醒后会引发EINTR错误（
                    # 取决于python的版本和具体的IOLoop实现，一般情况下通过 signal.set_wakeup_fd（）设置wakeup fd
                    # 来捕获信号进行处理，不引发InterruptedError[Raised when a system call is interrupted by
                    # an incoming signal. Corresponds to errno EINTR.]）。
                    #
                    # 这种会导致当前进程（线程）进入阻塞的系统调用被称为慢系统调用(slow system call)，比如accept、
                    # read、write、select、和open之类的函数。
                    if errno_from_exception(e) == errno.EINTR:
                        continue
                    else:
                        raise

                # 设置定时器以便在I/O循环阻塞超过预期时间时发送 SIGALRM 信号。
                #
                # signal.setitimer函数，提供三种定时器，它们相互独立，任意一个定时完成都将发送定时信号到进程，并且自动重新计时。
                # 1、ITIMER_REAL发送 SIGALRM，定时真实时间，与alarm类型相同。
                # 2、ITIMER_VIRT发送 SIGVTALRM，定时进程在用户态下的实际执行时间。
                # 3、ITIMER_PROF发送SIGPROF，定时进程在用户态和核心态下的实际执行时间。
                if self._blocking_signal_threshold is not None:
                    signal.setitimer(signal.ITIMER_REAL,
                                     self._blocking_signal_threshold, 0)

                # Pop one fd at a time from the set of pending fds and run
                # its handler. Since that handler may perform actions on
                # other file descriptors, there may be reentrant calls to
                # this IOLoop that update self._events
                #
                # 由于一个handler可能会操作其他文件描述符与IOLoop进行交互，比如调用
                # IOLoop.remove_handler方法等将导致self._events被修改。所以使用
                # while循环而不是for循环（要求迭代期间self._events不能被修改）。
                self._events.update(event_pairs)
                while self._events:
                    fd, events = self._events.popitem()
                    try:
                        fd_obj, handler_func = self._handlers[fd]
                        handler_func(fd_obj, events)
                    except (OSError, IOError) as e:
                        if errno_from_exception(e) == errno.EPIPE:
                            # Happens when the client closes the connection
                            pass
                        else:
                            self.handle_callback_exception(self._handlers.get(fd))
                    except Exception:
                        self.handle_callback_exception(self._handlers.get(fd))
                fd_obj = handler_func = None

        finally:
            # reset the stopped flag so another start/stop pair can be issued
            #
            # I/O循环结束重置_stopped状态，清理定时器，将当前IOLoop实例从当前线程移除绑定。
            self._stopped = False
            if self._blocking_signal_threshold is not None:
                signal.setitimer(signal.ITIMER_REAL, 0, 0)
            IOLoop._current.instance = old_current
            if old_wakeup_fd is not None:
                signal.set_wakeup_fd(old_wakeup_fd)
```

`start` 方法调用后IOLoop进入I/O主循环，要停止主循环只需调用`stop`方法。`stop`方法会将I/O循环设置为正在结束状态_running=False，_stopped=True，为了防止主循环已进入poll等待（sleep,没有就绪的文件描述符）而调用`self._waker.wake()`将主循环进程唤醒（self._waker包装的文件描述符 READ 事件已经在 initialize 方法中加入I/O循环。）。
```python
    def stop(self):
        self._running = False
        self._stopped = True
        self._waker.wake()

    def initialize(self, impl, time_func=None):
        super(PollIOLoop, self).initialize()

        [...省略部分代码...]

        # Create a pipe that we send bogus data to when we want to wake
        # the I/O loop when it is idle
        self._waker = Waker()
        self.add_handler(self._waker.fileno(),
                         lambda fd, events: self._waker.consume(),
                         self.READ)
```

`Waker` 内部创建了一个没有名字的管道和对应的处理器(Waker.consume)，IOLoop.initialize把管道的一端(Waker.fileno)放在了轮询文件描述符列表中。当需要停止时，在管道的另一端(Waker.write_fileno)随便写点什么(Waker.wake)便立即将主循环从poll等待中唤醒。
```python
class Waker(interface.Waker):
    def __init__(self):
        r, w = os.pipe()
        _set_nonblocking(r)
        _set_nonblocking(w)
        set_close_exec(r)
        set_close_exec(w)
        self.reader = os.fdopen(r, "rb", 0)
        self.writer = os.fdopen(w, "wb", 0)

    def fileno(self):
        return self.reader.fileno()

    def write_fileno(self):
        return self.writer.fileno()

    def wake(self):
        try:
            self.writer.write(b"x")
        except IOError:
            pass

    def consume(self):
        try:
            while True:
                result = self.reader.read()
                if not result:
                    break
        except IOError:
            pass

    def close(self):
        self.reader.close()
        self.writer.close()
```