# Linux 网络I/O模型

## 文件描述符

Linux 的内核将所有外部设备都看作一个文件来操作，对一个文件的读写操作会调用内核提供的系统命令，返回一个`file descriptor`(fd 文件描述符)。而对一个`socket` 的读写也会有相应的描述符，成为`socketfd` (socket描述符)，描述符就是一个数字，它指向内核中的一个结构体（文件路径，数据区等一些属性）。

## 用户空间（user space） 与 内核空间（kernel space）

User Space 是用户程序的运行空间；Kernel Space 是Linux 内核的运行空间。

为了安全起见，两者是隔离的，即使应用程序奔溃，内核也不受影响。

Kernel Space 可以执行任意命令，调用系统的一切资源；User Space只能执行简单的运算，不能直接调用系统资源。必须通过系统接口`system call`才能向内核发出指令

```java
str = "my string" // 用户空间
x = x + 2 // 用户空间
file.write(str) // 切换到内核空间
y = x + 4 // 切换回用户空间
```

![](..\resource\linux-user-kernel-space.png)

## 进程的阻塞

正在执行的进程，由于期待某些事件的发生。如请求系统资源，等待某种操作的完成，新数据尚未到达或无新工作等。则由系统自动执行阻塞`原语`（Block），使自己运行状态变为阻塞状态，释放cpu 资源。进程的阻塞是进程自身的一种主动行为，因此，只有处于运行状态的进程，才能转换为阻塞状态。

## 进程切换

为了控制进程的执行，内核需要挂起cpu 上正在执行的进程，并恢复以前挂起的某个进程的执行。这种行为叫做进程切换。因此可以说，任何进程都是在操作系统内核的支持下运行的与内核紧密关联。进程之间的切换其实是需要消耗CPU 时间的。

## 缓存IO

缓存IO 又被称作标准IO,大多数文件系统的默认IO操作都是缓存IO.在Linux的缓存IO机制中,数据先从磁盘复制到 kernel space 的缓冲区,然后从 kernel space缓冲区复制到应用程序的地址空间.

- 读操作: 操作系统检查kernel space 缓冲区 是否有需要数据,若已经缓存,则直接返回缓存数据,否则从磁盘读取缓存到操作系统缓存中.
- 写操作: 将数据从用户空间复制到 kernel space 缓冲区.这时,对于用户来说,写操作已经完成.何时写到磁盘中由系统调度决定,除非显示调用了sync 同步命令.

缓存IO 的优点:

- 一定程度上分离了内核空间与用户空间,保护系统本身的运行安全.
- 减少物理读盘次数,提高性能

缓存IO 的缺点:

- 在缓存IO机制中,DMA方式可以将数据直接从磁盘读取到页缓存中,或者将数据从页缓存直接写回到磁盘上,而不能直接在应用程序地址空间 和 磁盘之间进行数据传输.这样数据需要在传输过程中需要在User Space, Kernel Space 缓存空间进行多次数据拷贝,这些数据拷贝操作所带来的CPU 以及内存开销是非常大的.

引出了`zero copy `.

## UNIX IO 模型

在linux 中,对于一次IO访问(read 为例).数据先回被拷贝到操作系统内核的缓冲区中,然后才会从操作系统内核的缓冲区拷贝到应用程序的地址空间.当一个read 操作发生时,会经历如下两个阶段:

- 等待数据准备就绪
- 将数据从内核拷贝到进程中.

根据UNIX 网络编程对IO模型的分类，UNIX 提供了五种IO 模型，分别如下：

### 阻塞IO模型(Blocking IO)

最常用的IO模型就是阻塞IO模型，缺省情况下，所有文件操作都是阻塞的。以套接字接口为例讲解此模型：在进程空间中调用`recvfrom`，其系统调用直到数据包到达且被复制到应用进程的缓冲区中或者发生错误时才返回，在此期间会一直等待，进程在从调用`recvfrom`开始到它返回的整段时间内都是被阻塞的，因此被称为阻塞IO 模型。

blocking IO 的特点就是在IO 执行的以下两个阶段都会block：

- 等待数据准备就绪							阻塞
- 将数据从内核拷贝到进程中             阻塞

![](..\resource\blocking-io.png)



### 非阻塞IO模型  （Non Blocking IO）

`recvfrom`从应用层到内核层的时候，如果该缓存区没有数据的话，就直接返回一个EWOULDBLOCK错误，一般都对非阻塞IO模型进行轮训检查这个状态，看内核是否有数据到来。

在这个模型中，IO操作函数将不断轮训 数据是否已经准备好，如果没有准备好继续轮训。这个不断轮训的过程中，消耗了大量的CPU 资源。

no-blocking IO 模型的特点是用户不断的主动询问kernel 数据是否准备好：

- 等待数据准备就绪					非阻塞
- 将数据从内核拷贝到进程中    阻塞

![](..\resource\no-blocking-io.png)

### IO复用模型(IO Multiplexing )

IO Multiplexing 就是我们常说的 Linux 提供的 select,poll,epoll 系统操作。对于select,poll 单个进程可以同时处理多个fd，阻塞在select 操作上，这样 select,poll 可以同时侦测多个fd 是否处于就绪状态（select,poll 顺序扫描fd 是否就绪，且支持的fd 数量有限）。对于epoll，epoll 使用基于事件驱动方式代替顺序扫描，因此性能更高，当有fd 准备就绪的时候，立即回调rollback。

当用户进程调用了 select ，那么整个进程会被block，而同时，kernel 会监控多个select 负责的socket，当其中任意一个 socket 准备好了数据，select 就会返回，这个时候用户在调用read 操作，将数据从kernel 读取到用户进程。

下图 与 Blocking IO 的图没太大不同。事实上 IO 多路复用多了添加监视socket，以及select 函数的额外操作，效率会更差（IO 多路复用使用两个系统调用select，recvfrom，阻塞IO 只会调用一次 recovrom）。IO 多路复用的优势就是可以使用一个进程同时处理多个socket 的IO 请求，而Blocking IO 模型 监听多个需要开启多个线程。

在并发量不高的情况下 Blocking IO 的性能要高于 IO Multiplexing ，IO Multiplexing 的优势在于能够处理更多的连接。

IO 多路复用是 Reactor 设计模式的经典体现。

IO Multiplexing 模型的特点：

- 等待数据准备就绪				阻塞
- 将数据从内核拷贝到进程     阻塞

![](..\resource\multiplex-io.png)

### 异步IO模型

用户进程告知kernel，启动某个aio read操作，然后继续处理其余事情。从kernel 角度，当发现一个aio read 操作后，首先会立即返回，不发生block。然后等到数据准备完成，将数据拷贝到用户内存后，会发送signal通知用户进程 aio read 操作已经结束。

异步IO 模型是Proactor 模式的经典实现（Reactor 模式的异步版本）

异步IO 模型中关注的事件的结束，而不是事件的开始发生。

异步IO模型的特点：

- 等待数据准备就绪				非阻塞
- 将数据从内核拷贝到进程     非阻塞

![](..\resource\async-io.png)

### 信号驱动IO模型

允许socket进行信号驱动IO，并通过系统调用`sigaction `执行一个信号处理函数（调用立即返回，进程继续工作，非阻塞）。当数据准备就绪，为该进程生成一个`SIGIO`信号，通过信号回调通知应用调用`recvfrom`读取数据，并通知主函数处理数据

使用不是很多。

![](..\resource\signal-io.png)

### 



## 参考

Netty 权威指南

[https://wenchao.ren/2019/03/unix-IO%E6%A8%A1%E5%9E%8B/](https://wenchao.ren/2019/03/unix-IO模型/)

https://blog.csdn.net/caiwenfeng_for_23/article/details/8458299