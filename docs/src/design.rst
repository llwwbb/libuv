
.. _design:

Design overview
===============
libuv是跨平台支持库，最初为Node.js所写。它是基于事件驱动的异步I/O模型设计的。
libuv is cross-platform support library which was originally written for `Node.js`_. It's designed
around the event-driven asynchronous I/O model.

.. _Node.js: https://nodejs.org

该库不仅提供对不同I/O轮询机制的简单抽象：handles和streams为套接字和其他实体提供高级抽象；还跨平台文件I/O和线程功能。
The library provides much more than a simple abstraction over different I/O polling mechanisms:
'handles' and 'streams' provide a high level abstraction for sockets and other entities;
cross-platform file I/O and threading functionality is also provided, amongst other things.

下图说明了组成libuv的不同部分及其涉及的子系统
Here is a diagram illustrating the different parts that compose libuv and what subsystem they
relate to:

.. image:: static/architecture.png
    :scale: 75%
    :align: center


Handles and requests
^^^^^^^^^^^^^^^^^^^^
libuv与事件循环结合，为用户提供两种抽象： handles和requests
libuv provides users with 2 abstractions to work with, in combination with the event loop:
handles and requests.
handles表示活动时能够执行某些操作的长生命周期对象。例如：
Handles represent long-lived objects capable of performing certain operations while active. Some examples:

- A prepare handle gets its callback called once every loop iteration when active.
- A TCP server handle that gets its connection callback called every time there is a new connection.

Requests represent (typically) short-lived operations. These operations can be performed over a
handle: write requests are used to write data on a handle; or standalone: getaddrinfo requests
don't need a handle they run directly on the loop.


The I/O loop
^^^^^^^^^^^^
I/O（或事件）循环是libuv的核心部分。它建立了所有I/O操作的内容并将其绑定到一个单独的线程上。
可以同时运行多个事件循环，只要每一个都在不同的线程上运行。
libuv的事件循环（或与此相关的其他任何涉及循环或handles的API）都不是线程安全的，除非另有说明。
The I/O (or event) loop is the central part of libuv. It establishes the content for all I/O
operations, and it's meant to be tied to a single thread. One can run multiple event loops
as long as each runs in a different thread. The libuv event loop (or any other API involving
the loop or handles, for that matter) **is not thread-safe** except where stated otherwise.
事件循环遵循通常的单线程异步I / O方法：所有（网络）I/O都在非阻塞套接字上执行，这些套接字在给定平台上以可用的最佳机制进行轮询：
作为循环迭代的一部分，循环会阻塞等待已添加到轮询器中的套接字上的I/O活动，并且回调会被触发，指示套接字状态（可读可写挂起）这样handles可以读、写或执行需要的I/O操作。
The event loop follows the rather usual single threaded asynchronous I/O approach: all (network)
I/O is performed on non-blocking sockets which are polled using the best mechanism available
on the given platform: epoll on Linux, kqueue on OSX and other BSDs, event ports on SunOS and IOCP
on Windows. As part of a loop iteration the loop will block waiting for I/O activity on sockets
which have been added to the poller and callbacks will be fired indicating socket conditions
(readable, writable hangup) so handles can read, write or perform the desired I/O operation.
为了更好的理解事件循环的运行方式，下图说明了一次循环迭代中的所有阶段。
In order to better understand how the event loop operates, the following diagram illustrates all
stages of a loop iteration:

.. image:: static/loop_iteration.png
    :scale: 75%
    :align: center

循环概念"now"已更新。为了减少与时间相关的系统调用的数量，事件循环在事件循环tick的开始处缓存当前时间。
#. The loop concept of 'now' is updated. The event loop caches the current time at the start of
   the event loop tick in order to reduce the number of time-related system calls.
如果循环是活动的，则开始一次迭代，否则循环会立即退出。那么什么时候循环会被认为是活动的？
如果循环有活动并引用的handles，活动的请求或者关闭中的句柄，那么它就被认为是活动的。
#. If the loop is *alive*  an iteration is started, otherwise the loop will exit immediately. So,
   when is a loop considered to be *alive*? If a loop has active and ref'd handles, active
   requests or closing handles it's considered to be *alive*.
到期的计时器执行。所有预设时间在循环概念now之前的活动计时器，其回调会被调用。
#. Due timers are run. All active timers scheduled for a time before the loop's concept of *now*
   get their callbacks called.
待处理状态的回调被调用。大多数情况下，所有的I/O回调在I/O轮询后会被立即调用。
然而在某些情况下，此类回调会被推迟到下一个循环迭代。如果上一个循环迭代推迟了任何I/O回调，它将在此时运行。
#. Pending callbacks are called. All I/O callbacks are called right after polling for I/O, for the
   most part. There are cases, however, in which calling such a callback is deferred for the next
   loop iteration. If the previous iteration deferred any I/O callback it will be run at this point.
空闲handle回调被调用。尽管名称很不幸，但如果空闲handles处于活动状态，则每个循环迭代中都会运行它们。
#. Idle handle callbacks are called. Despite the unfortunate name, idle handles are run on every
   loop iteration, if they are active.
准备handle回调被调用。准备handle的回调会在循环为I/O而阻塞之前调用其回调。
#. Prepare handle callbacks are called. Prepare handles get their callbacks called right before
   the loop will block for I/O.
计算轮询超时时间。在循环为I/O而阻塞之前，计算阻塞的时间。这些是计算时间的规则。
#. Poll timeout is calculated. Before blocking for I/O the loop calculates for how long it should
   block. These are the rules when calculating the timeout:

         如果循环使用UV_RUN_NOWAIT标志运行，则超时时间为0.
        * If the loop was run with the ``UV_RUN_NOWAIT`` flag, the timeout is 0.
        如果循环将要停止（uv_stop被调用），超时为0.
        * If the loop is going to be stopped (:c:func:`uv_stop` was called), the timeout is 0.
        如果没有活动的handles或requests，超时为0.
        * If there are no active handles or requests, the timeout is 0.
        如果有活动的空闲handles，超时为0
        * If there are any idle handles active, the timeout is 0.
        如果有等待关闭的handles，超时为0
        * If there are any handles pending to be closed, the timeout is 0.
        如果以上情况均不匹配，则会采用最接近的计时器超时时间，或者如果没有活动的计时器，超时为无穷大。
        * If none of the above cases matches, the timeout of the closest timer is taken, or
          if there are no active timers, infinity.

循环为I/O而阻塞。此时循环会为I/O而阻塞上一步计算的时间。所有与I/O相关的，为读或写操作监听文件描述符的handles在此时调用其回调。
#. The loop blocks for I/O. At this point the loop will block for I/O for the duration calculated
   in the previous step. All I/O related handles that were monitoring a given file descriptor
   for a read or write operation get their callbacks called at this point.
检查handle回调被调用。检查handles在循环为I/O而阻塞后立即调用其回调。检查handle本质上是准备handle的对应。
#. Check handle callbacks are called. Check handles get their callbacks called right after the
   loop has blocked for I/O. Check handles are essentially the counterpart of prepare handles.
关闭回调被调用。如果handle因调用:c:func:`uv_close`被关闭，它会调用关闭回调。
#. Close callbacks are called. If a handle was closed by calling :c:func:`uv_close` it will
   get the close callback called.
特殊情况。如果循环使用``UV_RUN_ONCE``运行，那么意味着进一步的过程。很可能在为I/O阻塞后没有回调被执行，但过了一些时间，因此可能有一些计时器到期了，这些计时器的回调会被调用。
#. Special case in case the loop was run with ``UV_RUN_ONCE``, as it implies forward progress.
   It's possible that no I/O callbacks were fired after blocking for I/O, but some time has passed
   so there might be timers which are due, those timers get their callbacks called.
迭代结束。如果循环使用UV_RUN_NOWAIT或UV_RUN_ONCE模式运行，则迭代结束，并且uv_run（）将返回。
如果循环是使用UV_RUN_DEFAULT运行的，那么如果循环仍然是活动的，它将从头开始继续，否则也会结束。
#. Iteration ends. If the loop was run with ``UV_RUN_NOWAIT`` or ``UV_RUN_ONCE`` modes the
   iteration ends and :c:func:`uv_run` will return. If the loop was run with ``UV_RUN_DEFAULT``
   it will continue from the start if it's still *alive*, otherwise it will also end.

重点
.. important::
   libuv使用线程池使异步文件I / O操作成为可能，但是网络I/O**总是**在单个线程中执行，每个循环的线程。
    libuv uses a thread pool to make asynchronous file I/O operations possible, but
    network I/O is **always** performed in a single thread, each loop's thread.

.. note::
尽管轮询机制不同，但libuv使执行模型在Unix系统和Windows之间保持一致。
    While the polling mechanism is different, libuv makes the execution model consistent
    across Unix systems and Windows.

文件I/O
File I/O
^^^^^^^^
不同于网络I/O，libuv没有特定平台的文件I/O原语可以依赖，所以现在的方法是在线程池中运行阻塞的文件I/O操作。
Unlike network I/O, there are no platform-specific file I/O primitives libuv could rely on,
so the current approach is to run blocking file I/O operations in a thread pool.
有关跨平台文件I / O情况的详尽说明，请查看此文章。
For a thorough explanation of the cross-platform file I/O landscape, checkout
`this post <https://blog.libtorrent.org/2012/10/asynchronous-disk-io/>`_.
目前libuv使用一个全局线程池，所有的循环可以排队工作。当前在此池上运行3种类型的操作：
libuv currently uses a global thread pool on which all loops can queue work. 3 types of
operations are currently run on this pool:

文件系统操作
    * File system operations
    DNS方法
    * DNS functions (getaddrinfo and getnameinfo)
    用户通过uv_queue_work（）指定的代码
    * User specified code via :c:func:`uv_queue_work`
警告
有关更多详细信息，请参见“线程池工作调度”部分，但请记住，线程池的大小非常有限。
.. warning::
    See the :c:ref:`threadpool` section for more details, but keep in mind the thread pool size
    is quite limited.
