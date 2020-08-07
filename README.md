# 面试面经总结
全栈 / IOS


## 进程和线程 进程通信

进程和线程的主要差别在于它们是不同的操作系统资源管理方式。进程有独立的地址空间，一个进程崩溃后，在保护模式下不会对其它进程产生影响，而线程只是一个进程中的不同执行路径。线程有自己的堆栈和局部变量，但线程之间没有单独的地址空间，一个线程死掉就等于整个进程死掉，所以多进程的程序要比多线程的程序健壮，但在进程切换时，耗费资源较大，效率要差一些。但对于一些要求同时进行并且又要共享某些变量的并发操作，只能用线程，不能用进程。

1. 简而言之,一个程序至少有一个进程,一个进程至少有一个线程.
2. 线程的划分尺度小于进程，使得多线程程序的并发性高。
3. 另外，进程在执行过程中拥有独立的内存单元，而多个线程共享内存，从而极大地提高了程序的运行效率。
4. 线程在执行过程中与进程还是有区别的。每个独立的线程有一个程序运行的入口、顺序执行序列和程序的出口。但是线程不能够独立执行，必须依存在应用程序中，由应用程序提供多个线程执行控制。
5. 从逻辑角度来看，多线程的意义在于一个应用程序中，有多个执行部分可以同时执行。但操作系统并没有将多个线程看做多个独立的应用，来实现进程的调度和管理以及资源分配。这就是进程和线程的重要区别。

IPC目的

1）数据传输：一个进程需要将它的数据发送给另一个进程，发送的数据量在一个字节到几兆字节之间。
2）共享数据：多个进程想要操作共享数据，一个进程对共享数据的修改，别的进程应该立刻看到。
3）通知事件：一个进程需要向另一个或一组进程发送消息，通知它（它们）发生了某种事件（如进程终止时要通知父进程）。
4）资源共享：多个进程之间共享同样的资源。为了作到这一点，需要内核提供锁和同步机制。
5）进程控制：有些进程希望完全控制另一个进程的执行（如Debug进程），此时控制进程希望能够拦截另一个进程的所有陷入和异常，并能够及时知道它的状态改变。

IPC方式包括：管道、系统IPC（信号量、消息队列、共享内存）和套接字（socket）

### 管道：
3种。管道是面向字节流，自带互斥与同步机制，生命周期随进程。 
1. 普通管道PIPE, 通常有两种限制,一是半双工,数据同时只能单向传输;二是只能在父子或者兄弟进程间使用.，
2. 命令流管道s_pipe: 去除了第一种限制,为全双工，可以同时双向传输，
3. 命名管道FIFO, 去除了第二种限制,可以在许多并不相关的进程之间进行通讯。
4. 无名管道：没有磁盘节点，仅仅作为一个内存对象，用完就销毁了。因此没有显示的打开过程，实际在创建时自动打开，并且生成内存iNode，其内存对象和普通文件的一致，所以读写操作用的同样的接口，但是专用的。因为不能显式打开（没有任何标示），所以只能用在父子进程，兄弟进程， 或者其他继承了祖先进程的管道文件对象的两个进程间使用【具有共同祖先的进程】


### 信号量：

1. 临界资源：同一时刻，只能被一个进程访问的资源
2. 临界区：访问临界资源的代码区
3. 原子操作：任何情况下不能被打断的操作。

它是一个计数器，记录资源能被多少个进程同时访问。用于控制多进程对临界资源的访问（同步)），并且是非负值。主要作为进程间以及同一进程的不同线程间的同步手段。
操作：创建或获取，若是创建必须初始化，否则不用初始化。
```
int semget((key_t)key, int nsems, int flag);//创建或获取信号量

int semop(int semid, stuct sembuf*buf, int length);//加一操作（V操作）：释放资源；减一操作（P操作）：获取资源

int semct(int semid, int pos, int cmd);//初始化和删除
```

注：我们可以封装成库，实现信号量的创建或初始化，p操作，V操作，删除操作。。

### 消息队列：

消息队列是消息的链表，是存放在内核中并由消息队列标识符标识。因此是随内核持续的，只有在内核重起或者显示删除一个消息队列时，该消息队列才会真正被删除。。消息队列克服了信号传递信息少，管道只能承载无格式字节流以及缓冲区受限等特点。允许不同进程将格式化的数据流以消息队列形式发送给任意进程，对消息队列具有操作权限的进程都可以使用msgget完成对消息队列的操作控制，通过使用消息类型，进程可以按顺序读信息，或为消息安排优先级顺序。

与信号量相比，都以内核对象确保多进程访问同一消息队列。但消息队列发送实际数据，信号量进行进程同步控制。

与管道相比，管道发送的数据没有类型，读取数据端无差别从管道中按照前后顺序读取；消息队列有类型，读端可以根据数据类型读取特定的数据。

操作：创建或获取消息队列， int msgget((key_tkey, int flag);//若存在获取，否则创建它

发送消息：int msgsnd(int msgid, void *ptr, size_t size, int flag); ptr指向一个结构体存放类型和数据 size 数据的大小

接受消息：int msgrcv(int msgid, void *ptr, size_t size, long type, int flag);

删除消息队列： int msgctl(int msgid, int cmd, struct msgid_ds*buff);
       
### 共享内存：

共享内存就是映射一段能被其他进程所访问的内存，这段共享内存由一个进程创建，但多个进程都可以访问。    
共享内存是最快的一种IPC，因为不需要在客户进程和服务器进程之间赋值。使用共享内存的唯一注意的是是多个进程对一给定的存储区的同步访问。【若服务器进程正在向共享存储区写入数据，则写完数据之前客户进程不应读取数据，或者客户进程正在从共享内存中读取数据，服务进程不应写入数据。。所以我们要对共享内存进行同步控制，通常是信号量。】
```
int shmget((key_t)key, size_t size, int flag); //size 开辟内存空间的大小，flag：若存在则获取，否则创建共享内存存储段，返回一个标识符。

void *shmat(int shmid, void *addr, int flag); //将共享内存段连接到进程的地址空间中，返回一个共享内存首地址
```
函数shmat将标识号为shmid共享内存映射到调用进程的地址空间中，映射的地址由参数shmaddr和shmflg共同确定，其准则为：

1. 如果参数shmaddr取值为NULL，系统将自动确定共享内存链接到进程空间的首地址。

2. 如果参数shmaddr取值不为NULL且参数shmflg没有指定SHM_RND标志，系统将运用地址shmaddr链接共享内存。

3. 如果参数shmaddr取值不为NULL且参数shmflg指定了SHM_RND标志位，系统将地址shmaddr对齐后链接共享内存。其中选项SHM_RND的意思是取整对齐，常数SHMLBA代表了低边界地址的倍数，公式“shmaddr – (shmaddr % SHMLBA)”的意思是将地址shmaddr移动到低边界地址的整数倍上。
```
int shmdt(void *ptr); //断开进程与共享内存的链接
```
进程脱离共享内存区后，数据结构 shmid_ds 中的 shm_nattch 就会减 1 。但是共享段内存依然存在，只有 shm_attch 为 0 后，即没有任何进程再使用该共享内存区，共享内存区才在内核中被删除。一般来说，当一个进程终止时，它所附加的共享内存区都会自动脱离。
```
int shmctl(int shmid, int cmd, struct shmid_ds *buff); //删除共享内存（内核对象）
```
shmid是shmget返回的标识符；

cmd是执行的操作：有三种值，一般为 IPC_RMID  删除共享内存段；

buff默认为0.

如果共享内存已经与所有访问它的进程断开了连接，则调用IPC_RMID子命令后，系统将立即删除共享内存的标识符，并删除该共享内存区，以及所有相关的数据结构；

如果仍有别的进程与该共享内存保持连接，则调用IPC_RMID子命令后，该共享内存并不会被立即从系统中删除，而是被设置为IPC_PRIVATE状态，并被标记为”已被删除”（使用ipcs命令可以看到dest字段）；直到已有连接全部断开，该共享内存才会最终从系统中消失。

需要说明的是：一旦通过shmctl对共享内存进行了删除操作，则该共享内存将不能再接受任何新的连接，即使它依然存在于系统中！所以，可以确知， 在对共享内存删除之后不可能再有新的连接，则执行删除操作是安全的；否则，在删除操作之后如仍有新的连接发生，则这些连接都将可能失败！

### socket通信

适合同一主机的不同进程间和不同主机的进程间进行全双工网络通信。但并不只是Linux有，在所有提供了TCP/IP协议栈的操作系统中几乎都提供了socket，而所有这样操作系统，对套接字的编程方法几乎是完全一样的,即“网络编程”。

## 死锁产生 死锁避免
## 网络七层协议
## tcp udp属于哪层 区别
## 两次握手行不行
## tcp怎样保证可靠交付
## 输入url后的事件流程
## 路由表查找
## 获取mac地址的过程
## Hashmap底层实现
## 面向对象和面向过程
## 10亿数据查找目标，bitmap
## 浏览器渲染页面过程
## 设计一个StringBuffer
## UDP怎么才能实现可靠
## 设计多线程下载一个1G文件
## 算法:股票交易 
