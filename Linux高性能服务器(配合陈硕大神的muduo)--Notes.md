# 高性能服务器程序框架

## 1.服务器模型

### 1.1 C/S模型

C/S就是我们说的 Client/Server模型，所有客户端通过访问服务器端获取所需资源。

![1643178974971](C:\Users\user\AppData\Roaming\Typora\typora-user-images\1643178974971.png)

### 1.2 P2P模型

peer to peer，点对点模型，此模型也更符合网络通信的实际情况。P2P模型的缺点就是当用户之间的传输过多时，网络负载会变得很重。还有一个显著问题就是主机之间相互的寻找发现比较困难，为此我们会在初始的P2P模型中增加一个"发现服务器"，此服务器可以提供查找、提供资源等服务。

![1643179158154](C:\Users\user\AppData\Roaming\Typora\typora-user-images\1643179158154.png)



## 2.服务器编程框架

简单来说，框架如下图所示:

![1643179205710](C:\Users\user\AppData\Roaming\Typora\typora-user-images\1643179205710.png)

每个单元的意义对于"单个服务器"和"服务器集群"而言是不同的，但又是相通的。

对于单个服务器程序，I/O处理单元用于处理客户连接、读写网络数据。但是数据的收发未必在I/O单元进行，也可能在逻辑单元进行。对于服务器集群，I/O处理单元是一个专门接入的服务器程序，他可以选择负载最小的服务器工作以达到负载均衡。

一个逻辑单元通常是一个进程或者线程，他分析并处理客户数据，然后将结果传递给I/O处理单元或者直接传递给客户。对于服务器集群而言，一个逻辑单元就是一台逻辑服务器。

网络存储单元可以使数据库、缓存或者是本地文件，甚至一台服务器，这个模块并不是必须的比如ssh、telnet。

请求队列是各单元间通信的抽象，I/O模块收到连接请求后需要通知逻辑单元来处理该请求。同样，多个逻辑单元竞争访问资源时需要某种条件来进行调节**(见OS的死锁、进程线程同步等概念)**，其经常作为"池"的一部分被实现。对于服务器集群，网络存储单元则是各服务器之间建立好的永久的TCP连接。



## 3. I/O 模型

### 3.1.我们来探讨阻塞的问题。

socket被创建时都是默认"阻塞"的。其实对于所有文件描述符(fd)都有"阻塞"的概念。对于"阻塞I/O"，执行的系统调用可能因为无法立即完成而被挂起；反观"非阻塞I/O"，系统调用总是立即返回只不过是返回值不同罢了，未完成的任务可能返回-1，错误可见"errno"处。

我们在编写简单的网络程序时"阻塞"现象很常见。客户端通过"connect"系统调用连接服务器，connect发送同步报文给服务器并等待确认报文段，如果确认报文段没有立即到达客户端那么connect就会被挂起。我们在进行read时也是一种阻塞的表现，当用户缓冲区为空时什么也读不到就阻塞着，send、recv同理。

显然的，在事情已经发生的情况下操作非阻塞I/O能提高程序效率。因此，非阻塞I/O通常配合其他I/O通知机制一起使用，比如I/O复用和SIGIO信号。

I/O复用指的是通过将一组事件注册到内核中，内核会通过此函数把其中就绪的时间告知给应用程序。Linux上常用的I/O复用函数有select、poll、epoll_wait。但I/O复用函数本身是阻塞的，但他们具有同时监听多个I/O事件的能力。

SIGIO信号是为目标文件描述符指定一个宿主进程，被指定的进程就能捕捉信号。当目标文件描述符上有事件发生时，SIGIO信号的信号处理函数就会被触发就可以在信号处理函数中对文件描述符进行非阻塞IO操作了。

### 3.2.讨论同步和异步的问题。

我们上面讲到的阻塞I/O、信号驱动I/O、I/O复用都是同步I/O机制。因为I/O的读写操作都是发生在I/O事件之后，且由应用程序来完成。

对于异步I/O而言，用户可以直接对I/O进行读写操作，这些操作告诉给内核用户读写缓冲区的位置以及I/O操作完成后以何种方式通知应用程序。异步操作总是立即返回的因为真正的读写操作已经由内核接管。

有一种比较好辨别的方式是同步I/O向应用程序发出的是I/O就绪事件，而异步I/O发出的是完成事件。Linux下aio.h头文件提供了对异步I/O的支持。



## 4.两种高效的事件处理模式

### 4.1 Reactor模式

根据下图参考：(使用同步I/O模型，以epoll_wait为例)

![1643202584635](C:\Users\user\AppData\Roaming\Typora\typora-user-images\1643202584635.png)

1. 主线程往epoll内核注册socket的读就绪事件。
2. 主线程调用epoll_wait等待socket上有可读数据。
3. socket上有可读数据时epoll_wait通知主线程，主线程将此事件放入请求队列。
4. 睡眠在请求队列的某个工作线程被唤醒，从socket读数据，处理客户请求并往epoll注册socket写就绪事件。
5. 主线程调用epoll_wait等待socket可写。
6. 当socket可写时，epoll_wait通知主线程。主线程则再次将此事件插入队列。
7. 睡眠在请求队列的某个工作线程被唤醒，往socket上写服务器处理客户请求的结果。



**这里就能看出我们同步I/O的概念，我们注册完事件，当这个事件发生时通知给到的只是一个"就绪信号"罢了，最后还是需要分配工作线程去实际完成读写！**



### 4.2 Proactor模式

仍旧参考下图:(使用异步I/O模型)

![1643203080904](C:\Users\user\AppData\Roaming\Typora\typora-user-images\1643203080904.png)

1. 主线程调用aio_read函数向内核注册socket读完成事件，并告知内核用户读缓冲区位置，以及读操作完成时如何通告应用程序。
2. 主线程继续处理其他逻辑。
3. 当socket上的数据被读入用户缓冲区后内核用户会发送一个信号，通知应用程序数据已经可用。
4. 应用程序预先定义好信号处理函数并选择一个工作线程处理客户请求。工作线程处理完客户请求后，调用aio_write向内核注册socket写完成事件，并告知内核用户写缓冲区位置、操作完成时如何通知应用程序。
5. 主线程继续处理其他逻辑。
6. 当用户缓冲区数据被写入socket后，内核发送信号通知应用数据已经发送完毕。
7. 应用程序预先定义好的信号处理函数选择一个工作线程做善后处理，比如是否关闭socket等。

**主线程中的epoll_wait调用仅用于检测监听连接请求事件，不能用于检测连接socket的读写操作。**



## 5.两种高效的并发模式

并发，concurrency，不具体介绍何为并发，我们在OS和Database中已经学习过诸如多进程、多线程编程和关于锁的一些概念。并发是一个可以提高程序效率的方法，前提是认真对待你的并发程序，确保无BUG。

### 1. 半同步/半异步模式

这里我们还是需要强调一下同步异步的概念，因为这不同于上面的I/O模型的同步异步**(指返回一个事件就绪信号或者事件完成信号)**。

这里的同步指程序完全按照代码的顺序执行，异步则是指程序的执行由系统事件来驱动。常见的系统事件有中段、信号等，下面是一张同步和异步"读"操作的示意图：

![1643273397541](C:\Users\user\AppData\Roaming\Typora\typora-user-images\1643273397541.png)



在半同步/半异步模式下同步模型用于处理客户逻辑，相当于"编程框架"中的逻辑单元。异步线程就用于处理I/O请求，相当于框架里的I/O单元。

异步线程监听客户请求后将其封装成请求对象进入队列，请求队列通知某个工作在同步模式下的工作线程读取并处理该对象。流程图如下所示：

![1643273656634](C:\Users\user\AppData\Roaming\Typora\typora-user-images\1643273656634.png)

外部I/O源->被异步线程接收连接->异步线程工作封装请求事件为对象入队->按照分配算法得到一个同步线程或者请求队列的对象->同步线程工作，读取并处理请求    --------------》然后返回来，一步一步读取。

#### 1.半同步/半异步模式的变体-------半同步/半反应堆模式

流程图如下所示：

![1643274041804](C:\Users\user\AppData\Roaming\Typora\typora-user-images\1643274041804.png)

主线程充当异步线程调用epoll_wait来监听socket事件。当有新的socket连接到来，主线程就往epoll内核注册读写事件，一旦有读写事件发生就将该连接socket插入到请求队列中。所有工作线程通过竞争获得接管权。



#### 2.相对高效的半同步/半异步模式

流程图如下：它能使每个工作线程都同时处理多个客户连接

![1643274395072](C:\Users\user\AppData\Roaming\Typora\typora-user-images\1643274395072.png)



主线程只负责监听socket，socket的连接由工作线程负责。当有新的连接产生时，主线程就把新返回的socket发给某个工作线程**(可以通过管道通信的方式)**，此后工作线程负责该新socket上的任何I/O操作。

工作线程感知到管道数据是新的连接后就往自己的epoll内核里注册socket读写事件。

因此上图中，每个线程都是维护自己的事件循环，独立监听不同事件，实际上每个线程都是用异步模式在工作**（并非严格的半同步半异步模式）**。



### 2.领导者/追随者模式

流程图如下所示：

![1643275027033](C:\Users\user\AppData\Roaming\Typora\typora-user-images\1643275027033.png)