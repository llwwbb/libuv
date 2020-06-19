Basics of libuv
===============

libuv enforces an **asynchronous**, **event-driven** style of programming.  Its
core job is to provide an event loop and callback based notifications of I/O
and other activities.  libuv offers core utilities like timers, non-blocking
networking support, asynchronous file system access, child processes and more.

Event loops
-----------
在事件驱动的编程中，应用程序对特定事件感兴趣并当其发生时做出响应。从操作系统收集事件或监控其他事件资源的责任是由libuv处理的，
用户可以注册事件发生时要调用的回调。事件循环通常永远保持运行。伪代码表示如下：
In event-driven programming, an application expresses interest in certain events
and respond to them when they occur. The responsibility of gathering events
from the operating system or monitoring other sources of events is handled by
libuv, and the user can register callbacks to be invoked when an event occurs.
The event-loop usually keeps running *forever*. In pseudocode:

.. code-block:: python

    while there are still events to process:
        e = get the next event
        if there is a callback associated with e:
            call the callback
一些事件的例子：
Some examples of events are:
准备好写文件
* File is ready for writing
套接字数据读取就绪
* A socket has data ready to be read
计时器超时
* A timer has timed out
事件循环由 ``uv_run()``封装--使用libuv时的最终功能函数。
This event loop is encapsulated by ``uv_run()`` -- the end-all function when using
libuv.
系统程序最常见的活动是处理输入输出，而不是大量的数字运算。
使用常规输入输出函数的问题是它们 阻塞。
与处理器速度相比，实际写入硬盘或从网络读取操作耗费了不成比例的长时间。
函数在完成任务之前不会返回，因此你的程序什么都做不了
对于需要高性能的程序，这是主要障碍，而其他活动和其他I/O操作一直在等待。
The most common activity of systems programs is to deal with input and output,
rather than a lot of number-crunching. The problem with using conventional
input/output functions (``read``, ``fprintf``, etc.) is that they are
**blocking**. The actual write to a hard disk or reading from a network, takes
a disproportionately long time compared to the speed of the processor. The
functions don't return until the task is done, so that your program is doing
nothing. For programs which require high performance this is a major roadblock
as other activities and other I/O operations are kept waiting.
标准解决方案之一是使用线程。每个阻塞的I/O操作均在单独的线程（或线程池）中启动。当阻塞函数在线程中被调用时，处理器可以安排另一个线程运行，这实际上需要CPU。
One of the standard solutions is to use threads. Each blocking I/O operation is
started in a separate thread (or in a thread pool). When the blocking function
gets invoked in the thread, the processor can schedule another thread to run,
which actually needs the CPU.
libuv遵循的方法使用另一种样式，异步非阻塞样式。大部分现代操作系统提供了事件通知子系统。
例如，在套接字上的常规读调用会阻塞直到发送方实际发送了一些东西为止。
相反，应用程序可以请求操作系统监听套接字，并把事件通知放在队列中。
应用程序可以在方便的时候检查事件（也许在最大利用处理器之前进行一些数字运算。
它是异步的因为应用程序在一个时刻表达了兴趣，然后在另一个时刻（时间和空间上）使用数据。
它是非阻塞的因为应用程序进程可以自由的执行其他任务。
这很好的契合了libuv的事件循环方法，因为操作系统事件可以视为另一个libuv事件。
非阻塞确保其他事件可以跟它们进入一样快的被继续处理。
The approach followed by libuv uses another style, which is the **asynchronous,
non-blocking** style. Most modern operating systems provide event notification
subsystems. For example, a normal ``read`` call on a socket would block until
the sender actually sent something. Instead, the application can request the
operating system to watch the socket and put an event notification in the
queue. The application can inspect the events at its convenience (perhaps doing
some number crunching before to use the processor to the maximum) and grab the
data. It is **asynchronous** because the application expressed interest at one
point, then used the data at another point (in time and space). It is
**non-blocking** because the application process was free to do other tasks.
This fits in well with libuv's event-loop approach, since the operating system
events can be treated as just another libuv event. The non-blocking ensures
that other events can continue to be handled as fast as they come in [#]_.

.. NOTE::
    I/O在后台是如何运行的我们不关心，但是由于我们计算机硬件的工作方式（一线程作为处理器的基本单元）
    libuv和OS通常以非阻塞方式运行后台/工作线程并且/或者轮询执行任务
    How the I/O is run in the background is not of our concern, but due to the
    way our computer hardware works, with the thread as the basic unit of the
    processor, libuv and OSes will usually run background/worker threads and/or
    polling to perform tasks in a non-blocking manner.

Bert Belder, one of the libuv core developers has a small video explaining the
architecture of libuv and its background. If you have no prior experience with
either libuv or libev, it is a quick, useful watch.

libuv's event loop is explained in more detail in the `documentation
<http://docs.libuv.org/en/v1.x/design.html#the-i-o-loop>`_.

.. raw:: html

    <iframe width="560" height="315"
    src="https://www.youtube-nocookie.com/embed/nGn60vDSxQ4" frameborder="0"
    allowfullscreen></iframe>

Hello World
-----------

With the basics out of the way, let's write our first libuv program. It does
nothing, except start a loop which will exit immediately.

.. rubric:: helloworld/main.c
.. literalinclude:: ../../code/helloworld/main.c
    :linenos:

This program quits immediately because it has no events to process. A libuv
event loop has to be told to watch out for events using the various API
functions.

Starting with libuv v1.0, users should allocate the memory for the loops before
initializing it with ``uv_loop_init(uv_loop_t *)``. This allows you to plug in
custom memory management. Remember to de-initialize the loop using
``uv_loop_close(uv_loop_t *)`` and then delete the storage. The examples never
close loops since the program quits after the loop ends and the system will
reclaim memory. Production grade projects, especially long running systems
programs, should take care to release correctly.

Default loop
++++++++++++
默认循环由libuv提供，可通过``uv_default_loop()``获得。
如果只需要一个循环，则应使用此循环。
A default loop is provided by libuv and can be accessed using
``uv_default_loop()``. You should use this loop if you only want a single
loop.

.. note::
    node.js使用默认循环作为其主循环。如果您正在编写绑定，则应注意这一点。
    node.js uses the default loop as its main loop. If you are writing bindings
    you should be aware of this.

.. _libuv-error-handling:
错误处理
Error handling
--------------
可能失败的初始化函数或同步函数会在错误时返回负数。可能失败的异步函数会将状态参数传递给其回调。错误消息定义为``UV_E*``常量。
Initialization functions or synchronous functions which may fail return a negative number on error. Async functions that may fail will pass a status parameter to their callbacks. The error messages are defined as ``UV_E*`` `constants`_. 

.. _constants: http://docs.libuv.org/en/v1.x/errors.html#error-constants
可以使用``uv_strerror(int)``和``uv_err_name(int)``函数分别获取错误的``const char *``类型的描述或错误的名字
You can use the ``uv_strerror(int)`` and ``uv_err_name(int)`` functions
to get a ``const char *`` describing the error or the error name respectively.
I/O读回调（例如文件和套接字）传递参数``nread``。如果``nread``小于0，则表示有错误。（UV_EOF是文件结尾错误，你可能需要额外处理）
I/O read callbacks (such as for files and sockets) are passed a parameter ``nread``. If ``nread`` is less than 0, there was an error (UV_EOF is the end of file error, which you may want to handle differently).

Handles and Requests
--------------------
用户使用libuv对特定事件表示兴趣。这通常通过创建一个对I/O设备，计时器或进程的handle来完成。
handles是名为``uv_TYPE_t``的不透明结构，其中TYPE表示该handle的用途。
libuv works by the user expressing interest in particular events. This is
usually done by creating a **handle** to an I/O device, timer or process.
Handles are opaque structs named as ``uv_TYPE_t`` where type signifies what the
handle is used for. 

.. rubric:: libuv watchers
.. code-block:: c

    /* Handle types. */
    typedef struct uv_loop_s uv_loop_t;
    typedef struct uv_handle_s uv_handle_t;
    typedef struct uv_dir_s uv_dir_t;
    typedef struct uv_stream_s uv_stream_t;
    typedef struct uv_tcp_s uv_tcp_t;
    typedef struct uv_udp_s uv_udp_t;
    typedef struct uv_pipe_s uv_pipe_t;
    typedef struct uv_tty_s uv_tty_t;
    typedef struct uv_poll_s uv_poll_t;
    typedef struct uv_timer_s uv_timer_t;
    typedef struct uv_prepare_s uv_prepare_t;
    typedef struct uv_check_s uv_check_t;
    typedef struct uv_idle_s uv_idle_t;
    typedef struct uv_async_s uv_async_t;
    typedef struct uv_process_s uv_process_t;
    typedef struct uv_fs_event_s uv_fs_event_t;
    typedef struct uv_fs_poll_s uv_fs_poll_t;
    typedef struct uv_signal_s uv_signal_t;

    /* Request types. */
    typedef struct uv_req_s uv_req_t;
    typedef struct uv_getaddrinfo_s uv_getaddrinfo_t;
    typedef struct uv_getnameinfo_s uv_getnameinfo_t;
    typedef struct uv_shutdown_s uv_shutdown_t;
    typedef struct uv_write_s uv_write_t;
    typedef struct uv_connect_s uv_connect_t;
    typedef struct uv_udp_send_s uv_udp_send_t;
    typedef struct uv_fs_s uv_fs_t;
    typedef struct uv_work_s uv_work_t;

handles表示长生命周期的对象。在这些对象上的异步操作使用**requests**来确定。
request是短生命周期的（通常只用于一个回调），并且通常表示handle上的一个I/O操作。
requests用来保留initiation和单个动作回调之间的上下文。
例如，一个UDP套接字由``uv_udp_t``表示，而对套接字的单独写入使用uv_udp_send_t结构，该结构在写入完成后传递给回调。
Handles represent long-lived objects. Async operations on such handles are
identified using **requests**. A request is short-lived (usually used across
only one callback) and usually indicates one I/O operation on a handle.
Requests are used to preserve context between the initiation and the callback
of individual actions. For example, an UDP socket is represented by
a ``uv_udp_t``, while individual writes to the socket use a ``uv_udp_send_t``
structure that is passed to the callback after the write is done.
handles由对应函数完成设置。
Handles are setup by a corresponding::

    uv_TYPE_init(uv_loop_t *, uv_TYPE_t *)

function.
回调是由libuv在观察者感兴趣的事件发生时调用的函数，应用程序的特定逻辑通常实现在回调用。
例如，IO观察者的回调会收到从文件中读取的数据，定时器回调会在超时时触发，等等。
Callbacks are functions which are called by libuv whenever an event the watcher
is interested in has taken place. Application specific logic will usually be
implemented in the callback. For example, an IO watcher's callback will receive
the data read from a file, a timer callback will be triggered on timeout and so
on.

Idling
++++++
这是使用空闲handle的例子。回调在事件循环的每一轮被调用一次。
在:doc:`utilities`中讨论了空闲handles的一个用例。
让我们使用空闲观察者来查看观察者的生命周期，并查看``uv_run()``是如何因为存在观察者而阻塞的。
当达到数量，观察者停止，``uv_run()``由于没有活动的观察者而停止。
Here is an example of using an idle handle. The callback is called once on
every turn of the event loop. A use case for idle handles is discussed in
:doc:`utilities`. Let us use an idle watcher to look at the watcher life cycle
and see how ``uv_run()`` will now block because a watcher is present. The idle
watcher is stopped when the count is reached and ``uv_run()`` exits since no
event watchers are active.

.. rubric:: idle-basic/main.c
.. literalinclude:: ../../code/idle-basic/main.c
    :emphasize-lines: 6,10,14-17
保存上下文
Storing context
+++++++++++++++
在基于回调的编程风格中，你经常希望在调用方和回调之间传递上下文--特定的应用信息--。
所有的handles和requests有``void* data``成员，你可以设置为上下文，在回调中回退。
这是整个C库生态系统中使用的常见模式。另外，uv_loop_t也具有类似的数据成员。
In callback based programming style you'll often want to pass some 'context' --
application specific information -- between the call site and the callback. All
handles and requests have a ``void* data`` member which you can set to the
context and cast back in the callback. This is a common pattern used throughout
the C library ecosystem. In addition ``uv_loop_t`` also has a similar data
member.

----

.. [#] Depending on the capacity of the hardware of course.
