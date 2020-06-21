文件系统
Filesystem
==========
简单的文件系统读/写由uv_fs_ *函数和uv_fs_t结构实现。
Simple filesystem read/write is achieved using the ``uv_fs_*`` functions and the
``uv_fs_t`` struct.

.. note::
    libuv文件系统操作不同于:doc:`socket operations <networking>`.
    套接字操作使用操作系统提供的的非阻塞操作。文件系统操作内部使用阻塞函数，但是在线程池中调用，
    在应用交互需要的时候通知注册在事件循环上的观察者。
    The libuv filesystem operations are different from :doc:`socket operations
    <networking>`. Socket operations use the non-blocking operations provided
    by the operating system. Filesystem operations use blocking functions
    internally, but invoke these functions in a `thread pool`_ and notify
    watchers registered with the event loop when application interaction is
    required.

.. _thread pool: http://docs.libuv.org/en/v1.x/threadpool.html#thread-pool-work-scheduling
所有文件系统功能都有两种形式-同步和异步。
All filesystem functions have two forms - *synchronous* and *asynchronous*.
同步模式会自动调用（并阻塞）如果回调为空。函数的返回值是:ref:`libuv error code <libuv-error-handling>`.
这通常是同步调用的唯一用途。异步模式会在传递回调并且返回值为0时调用。
The *synchronous* forms automatically get called (and **block**) if the
callback is null. The return value of functions is a :ref:`libuv error code
<libuv-error-handling>`. This is usually only useful for synchronous calls.
The *asynchronous* form is called when a callback is passed and the return
value is 0.
读/写文件
Reading/Writing files
---------------------
文件描述符可使用以下函数获取。
A file descriptor is obtained using

.. code-block:: c

    int uv_fs_open(uv_loop_t* loop, uv_fs_t* req, const char* path, int flags, int mode, uv_fs_cb cb)
``flags`` and ``mode``是标准unix flags
``flags`` and ``mode`` are standard
`Unix flags <http://man7.org/linux/man-pages/man2/open.2.html>`_.
libuv负责转换为适当的Windows flags。
libuv takes care of converting to the appropriate Windows flags.
文件描述符使用下面的函数关闭
File descriptors are closed using

.. code-block:: c

    int uv_fs_close(uv_loop_t* loop, uv_fs_t* req, uv_file file, uv_fs_cb cb)

文件系统回调签名如下
Filesystem operation callbacks have the signature:

.. code-block:: c

    void callback(uv_fs_t* req);
让我们看一个``cat``的简单实现。由注册一个打开文件的回调开始。
Let's see a simple implementation of ``cat``. We start with registering
a callback for when the file is opened:

.. rubric:: uvcat/main.c - opening a file
.. literalinclude:: ../../code/uvcat/main.c
    :linenos:
    :lines: 41-53
    :emphasize-lines: 4, 6-7
在``uv_fs_open``的回调中，``uv_fs_t``的``result``字段就是文件描述符。
如果文件成功打开，就开始读取。
The ``result`` field of a ``uv_fs_t`` is the file descriptor in case of the
``uv_fs_open`` callback. If the file is successfully opened, we start reading it.

.. rubric:: uvcat/main.c - read callback
.. literalinclude:: ../../code/uvcat/main.c
    :linenos:
    :lines: 26-40
    :emphasize-lines: 2,8,12
在读取调用中，你需要传入一个初始化好的buffer，在读回调触发前，会被填入数据。
``uv_fs_*``操作几乎直接映射到某些POSIX函数，所以在这种情况下EOF由``result``为0表示。
在流或管道的情况下，UV_EOF常数将作为状态传递。
In the case of a read call, you should pass an *initialized* buffer which will
be filled with data before the read callback is triggered. The ``uv_fs_*``
operations map almost directly to certain POSIX functions, so EOF is indicated
in this case by ``result`` being 0. In the case of streams or pipes, the
``UV_EOF`` constant would have been passed as a status instead.
这里你看到了异步编程的常见模式。``uv_fs_close()``调用是同步的。通常，一次完成，或者作为启动或关闭阶段完成的任务是同步的，
因为当程序执行其主要任务并处理多路I/O源时，我们才对快速I/O感兴趣。
对于单个任务，性能差异通常可以忽略不计，还可能使代码更简洁。
Here you see a common pattern when writing asynchronous programs. The
``uv_fs_close()`` call is performed synchronously. *Usually tasks which are
one-off, or are done as part of the startup or shutdown stage are performed
synchronously, since we are interested in fast I/O when the program is going
about its primary task and dealing with multiple I/O sources*. For solo tasks
the performance difference usually is negligible and may lead to simpler code.
使用``uv_fs_write()``进行文件写同样简单。写入完成后回调会被触发。
在我们的例子中，回调只是驱动下一次读取。 因此，读取和写入通过回调以锁步方式进行。
Filesystem writing is similarly simple using ``uv_fs_write()``.  *Your callback
will be triggered after the write is complete*.  In our case the callback
simply drives the next read. Thus read and write proceed in lockstep via
callbacks.

.. rubric:: uvcat/main.c - write callback
.. literalinclude:: ../../code/uvcat/main.c
    :linenos:
    :lines: 16-24
    :emphasize-lines: 6

.. warning::
    由于为提高性能而对文件系统和磁盘驱动器进行的配置，一个成功的写入可能尚未提交到磁盘。
    Due to the way filesystems and disk drives are configured for performance,
    a write that 'succeeds' may not be committed to disk yet.
我们设置多米诺骨牌在``main()``中滚动。
We set the dominos rolling in ``main()``:

.. rubric:: uvcat/main.c
.. literalinclude:: ../../code/uvcat/main.c
    :linenos:
    :lines: 55-
    :emphasize-lines: 2

.. warning::
在文件系统的请求中，必须始终调用``uv_fs_req_cleanup()``函数，来释放libuv中申请的内部内存。
    The ``uv_fs_req_cleanup()`` function must always be called on filesystem
    requests to free internal memory allocations in libuv.
文件系统操作
Filesystem operations
---------------------
所有标准文件系统操作像``unlink``, ``rmdir``, ``stat``都支持异步，有直观的参数顺序。
它们与读取/写入/打开调用遵循相同的模式，在uv_fs_t.result字段中返回结果。完整清单：
All the standard filesystem operations like ``unlink``, ``rmdir``, ``stat`` are
supported asynchronously and have intuitive argument order. They follow the
same patterns as the read/write/open calls, returning the result in the
``uv_fs_t.result`` field. The full list:

.. rubric:: Filesystem operations
.. code-block:: c

    int uv_fs_close(uv_loop_t* loop, uv_fs_t* req, uv_file file, uv_fs_cb cb);
    int uv_fs_open(uv_loop_t* loop, uv_fs_t* req, const char* path, int flags, int mode, uv_fs_cb cb);
    int uv_fs_read(uv_loop_t* loop, uv_fs_t* req, uv_file file, const uv_buf_t bufs[], unsigned int nbufs, int64_t offset, uv_fs_cb cb);
    int uv_fs_unlink(uv_loop_t* loop, uv_fs_t* req, const char* path, uv_fs_cb cb);
    int uv_fs_write(uv_loop_t* loop, uv_fs_t* req, uv_file file, const uv_buf_t bufs[], unsigned int nbufs, int64_t offset, uv_fs_cb cb);
    int uv_fs_copyfile(uv_loop_t* loop, uv_fs_t* req, const char* path, const char* new_path, int flags, uv_fs_cb cb);
    int uv_fs_mkdir(uv_loop_t* loop, uv_fs_t* req, const char* path, int mode, uv_fs_cb cb);
    int uv_fs_mkdtemp(uv_loop_t* loop, uv_fs_t* req, const char* tpl, uv_fs_cb cb);
    int uv_fs_rmdir(uv_loop_t* loop, uv_fs_t* req, const char* path, uv_fs_cb cb);
    int uv_fs_scandir(uv_loop_t* loop, uv_fs_t* req, const char* path, int flags, uv_fs_cb cb);
    int uv_fs_scandir_next(uv_fs_t* req, uv_dirent_t* ent);
    int uv_fs_opendir(uv_loop_t* loop, uv_fs_t* req, const char* path, uv_fs_cb cb);
    int uv_fs_readdir(uv_loop_t* loop, uv_fs_t* req, uv_dir_t* dir, uv_fs_cb cb);
    int uv_fs_closedir(uv_loop_t* loop, uv_fs_t* req, uv_dir_t* dir, uv_fs_cb cb);
    int uv_fs_stat(uv_loop_t* loop, uv_fs_t* req, const char* path, uv_fs_cb cb);
    int uv_fs_fstat(uv_loop_t* loop, uv_fs_t* req, uv_file file, uv_fs_cb cb);
    int uv_fs_rename(uv_loop_t* loop, uv_fs_t* req, const char* path, const char* new_path, uv_fs_cb cb);
    int uv_fs_fsync(uv_loop_t* loop, uv_fs_t* req, uv_file file, uv_fs_cb cb);
    int uv_fs_fdatasync(uv_loop_t* loop, uv_fs_t* req, uv_file file, uv_fs_cb cb);
    int uv_fs_ftruncate(uv_loop_t* loop, uv_fs_t* req, uv_file file, int64_t offset, uv_fs_cb cb);
    int uv_fs_sendfile(uv_loop_t* loop, uv_fs_t* req, uv_file out_fd, uv_file in_fd, int64_t in_offset, size_t length, uv_fs_cb cb);
    int uv_fs_access(uv_loop_t* loop, uv_fs_t* req, const char* path, int mode, uv_fs_cb cb);
    int uv_fs_chmod(uv_loop_t* loop, uv_fs_t* req, const char* path, int mode, uv_fs_cb cb);
    int uv_fs_utime(uv_loop_t* loop, uv_fs_t* req, const char* path, double atime, double mtime, uv_fs_cb cb);
    int uv_fs_futime(uv_loop_t* loop, uv_fs_t* req, uv_file file, double atime, double mtime, uv_fs_cb cb);
    int uv_fs_lstat(uv_loop_t* loop, uv_fs_t* req, const char* path, uv_fs_cb cb);
    int uv_fs_link(uv_loop_t* loop, uv_fs_t* req, const char* path, const char* new_path, uv_fs_cb cb);
    int uv_fs_symlink(uv_loop_t* loop, uv_fs_t* req, const char* path, const char* new_path, int flags, uv_fs_cb cb);
    int uv_fs_readlink(uv_loop_t* loop, uv_fs_t* req, const char* path, uv_fs_cb cb);
    int uv_fs_realpath(uv_loop_t* loop, uv_fs_t* req, const char* path, uv_fs_cb cb);
    int uv_fs_fchmod(uv_loop_t* loop, uv_fs_t* req, uv_file file, int mode, uv_fs_cb cb);
    int uv_fs_chown(uv_loop_t* loop, uv_fs_t* req, const char* path, uv_uid_t uid, uv_gid_t gid, uv_fs_cb cb);
    int uv_fs_fchown(uv_loop_t* loop, uv_fs_t* req, uv_file file, uv_uid_t uid, uv_gid_t gid, uv_fs_cb cb);
    int uv_fs_lchown(uv_loop_t* loop, uv_fs_t* req, const char* path, uv_uid_t uid, uv_gid_t gid, uv_fs_cb cb);


.. _buffers-and-streams:

Buffers and Streams
-------------------
在libuv中，基础I/O handle是流(``uv_stream_t``)。TCP套接字，UDP套接字，文件I/O管道和IPC都视为流的子类。
The basic I/O handle in libuv is the stream (``uv_stream_t``). TCP sockets, UDP
sockets, and pipes for file I/O and IPC are all treated as stream subclasses.
使用每个子类的自定义函数初始化流，然后使用下面的代码执行操作。
Streams are initialized using custom functions for each subclass, then operated
upon using

.. code-block:: c

    int uv_read_start(uv_stream_t*, uv_alloc_cb alloc_cb, uv_read_cb read_cb);
    int uv_read_stop(uv_stream_t*);
    int uv_write(uv_write_t* req, uv_stream_t* handle,
                 const uv_buf_t bufs[], unsigned int nbufs, uv_write_cb cb);
基于流的函数比文件系统函数更易于使用，当``uv_read_start()``被调用一次，libuv就会自动保持从流中读取数据，直到调用``uv_read_stop()``。
The stream based functions are simpler to use than the filesystem ones and
libuv will automatically keep reading from a stream when ``uv_read_start()`` is
called once, until ``uv_read_stop()`` is called.
数据的离散单位是buffer -- ``uv_buf_t``.它是一个指向bytes的指针(``uv_buf_t.base``)和长度(``uv_buf_t.len``)的简单集合。
``uv_buf_t``轻量级，值传递。需要管理的是实际的bytes，必须由应用程序来分配和释放。
The discrete unit of data is the buffer -- ``uv_buf_t``. This is simply
a collection of a pointer to bytes (``uv_buf_t.base``) and the length
(``uv_buf_t.len``). The ``uv_buf_t`` is lightweight and passed around by value.
What does require management is the actual bytes, which have to be allocated
and freed by the application.

.. ERROR::

    THIS PROGRAM DOES NOT ALWAYS WORK, NEED SOMETHING BETTER**

To demonstrate streams we will need to use ``uv_pipe_t``. This allows streaming
local files [#]_. Here is a simple tee utility using libuv.  Doing all operations
asynchronously shows the power of evented I/O. The two writes won't block each
other, but we have to be careful to copy over the buffer data to ensure we don't
free a buffer until it has been written.

The program is to be executed as::

    ./uvtee <output_file>

We start off opening pipes on the files we require. libuv pipes to a file are
opened as bidirectional by default.

.. rubric:: uvtee/main.c - read on pipes
.. literalinclude:: ../../code/uvtee/main.c
    :linenos:
    :lines: 61-80
    :emphasize-lines: 4,5,15

The third argument of ``uv_pipe_init()`` should be set to 1 for IPC using named
pipes. This is covered in :doc:`processes`. The ``uv_pipe_open()`` call
associates the pipe with the file descriptor, in this case ``0`` (standard
input).

We start monitoring ``stdin``. The ``alloc_buffer`` callback is invoked as new
buffers are required to hold incoming data. ``read_stdin`` will be called with
these buffers.

.. rubric:: uvtee/main.c - reading buffers
.. literalinclude:: ../../code/uvtee/main.c
    :linenos:
    :lines: 19-22,44-60

The standard ``malloc`` is sufficient here, but you can use any memory allocation
scheme. For example, node.js uses its own slab allocator which associates
buffers with V8 objects.

The read callback ``nread`` parameter is less than 0 on any error. This error
might be EOF, in which case we close all the streams, using the generic close
function ``uv_close()`` which deals with the handle based on its internal type.
Otherwise ``nread`` is a non-negative number and we can attempt to write that
many bytes to the output streams. Finally remember that buffer allocation and
deallocation is application responsibility, so we free the data.

The allocation callback may return a buffer with length zero if it fails to
allocate memory. In this case, the read callback is invoked with error
UV_ENOBUFS. libuv will continue to attempt to read the stream though, so you
must explicitly call ``uv_close()`` if you want to stop when allocation fails.

The read callback may be called with ``nread = 0``, indicating that at this
point there is nothing to be read. Most applications will just ignore this.

.. rubric:: uvtee/main.c - Write to pipe
.. literalinclude:: ../../code/uvtee/main.c
    :linenos:
    :lines: 9-13,23-42

``write_data()`` makes a copy of the buffer obtained from read. This buffer
does not get passed through to the write callback trigged on write completion. To
get around this we wrap a write request and a buffer in ``write_req_t`` and
unwrap it in the callbacks. We make a copy so we can free the two buffers from
the two calls to ``write_data`` independently of each other. While acceptable
for a demo program like this, you'll probably want smarter memory management,
like reference counted buffers or a pool of buffers in any major application.

.. WARNING::

    If your program is meant to be used with other programs it may knowingly or
    unknowingly be writing to a pipe. This makes it susceptible to `aborting on
    receiving a SIGPIPE`_. It is a good idea to insert::

        signal(SIGPIPE, SIG_IGN)

    in the initialization stages of your application.

.. _aborting on receiving a SIGPIPE: http://pod.tst.eu/http://cvs.schmorp.de/libev/ev.pod#The_special_problem_of_SIGPIPE

File change events
------------------

All modern operating systems provide APIs to put watches on individual files or
directories and be informed when the files are modified. libuv wraps common
file change notification libraries [#fsnotify]_. This is one of the more
inconsistent parts of libuv. File change notification systems are themselves
extremely varied across platforms so getting everything working everywhere is
difficult. To demonstrate, I'm going to build a simple utility which runs
a command whenever any of the watched files change::

    ./onchange <command> <file1> [file2] ...

The file change notification is started using ``uv_fs_event_init()``:

.. rubric:: onchange/main.c - The setup
.. literalinclude:: ../../code/onchange/main.c
    :linenos:
    :lines: 26-
    :emphasize-lines: 15

The third argument is the actual file or directory to monitor. The last
argument, ``flags``, can be:

.. code-block:: c

    /*
    * Flags to be passed to uv_fs_event_start().
    */
    enum uv_fs_event_flags {
        UV_FS_EVENT_WATCH_ENTRY = 1,
        UV_FS_EVENT_STAT = 2,
        UV_FS_EVENT_RECURSIVE = 4
    };

``UV_FS_EVENT_WATCH_ENTRY`` and ``UV_FS_EVENT_STAT`` don't do anything (yet).
``UV_FS_EVENT_RECURSIVE`` will start watching subdirectories as well on
supported platforms.

The callback will receive the following arguments:

  #. ``uv_fs_event_t *handle`` - The handle. The ``path`` field of the handle
     is the file on which the watch was set.
  #. ``const char *filename`` - If a directory is being monitored, this is the
     file which was changed. Only non-``null`` on Linux and Windows. May be ``null``
     even on those platforms.
  #. ``int flags`` - one of ``UV_RENAME`` or ``UV_CHANGE``, or a bitwise OR of
       both.
  #. ``int status`` - Currently 0.

In our example we simply print the arguments and run the command using
``system()``.

.. rubric:: onchange/main.c - file change notification callback
.. literalinclude:: ../../code/onchange/main.c
    :linenos:
    :lines: 9-24

----

.. [#fsnotify] inotify on Linux, FSEvents on Darwin, kqueue on BSDs,
               ReadDirectoryChangesW on Windows, event ports on Solaris, unsupported on Cygwin
.. [#] see :ref:`pipes`
