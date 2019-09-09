# 14.1 概述

本章讨论 I/O 的高级操作，首先是在 I/O 上设置超时，这里有三种方法。然后是 read 和 write 的三个变体：
1. recv 和 send 允许通过第四个参数从进程到内核传递标志
2. readv 和 writev 允许指定往其中输入数据或从其中输出数据的缓冲区向量
3. recvmsg 和 sendmsg 结合了其他 I/O 函数的所有特性，并具备接收和发送辅助数据的新能力

# 14.2 套接字超时

设置超时的方法有 3 种：
1. 调用 alarm，它在指定超时期满时产生 SIGALRM 信号。这个方法涉及信号处理，而信号处理在不同的实现上存在差异，而且可能干扰进程中现有的 alarm 调用
2. 在 select 中阻塞等待 I/O（select 有内置时间限制）以此代替直接阻塞在 read 或 write 调用上
3. 使用较新的 SO_RCVTIMEO 和 SO_SNDTIMEO 套接字选项。这个方法的问题在于并非所有实现都支持这两个套接字选项。

&emsp;&emsp;上述三个技术都适用于输入和输出操作（例如 read，write 以及诸如 recvfrom，sendto 之类），不过我们依然期待可用于 connect 的技术，因为 TCP 内置的 connect 超时相当长。select 可用来在 connect 上设置超时的先决条件是相应套接字处于非阻塞模式，而那两个套接字选项对 connect 并不适用。我们还指出，前两个技术使用于任何描述符，而第三个技术仅仅适用于套接字描述符（因为是套接字描述符选项）

### 1. 使用 SIGALRM 为 connect 设置超时

```cpp
static void	connect_alarm(int);

int connect_timeo(int sockfd, const SA *saptr, socklen_t salen, int nsec)
{
	Sigfunc	*sigfunc;
	int		n;
    // 设置 SIGALRM 信号的处理函数，并保存原有的处理函数到 sigfunc
	sigfunc = Signal(SIGALRM, connect_alarm);
    // 设置报警时钟的秒数，返回值是上一次设置的剩余秒数，没有就返回 0
	if (alarm(nsec) != 0)
		err_msg("connect_timeo: alarm was already set");
    // 调用 connect，调用中断就设置 error 设置为 TIMEOUT，并关闭套接字，防止三路握手继续进行
	if ( (n = connect(sockfd, saptr, salen)) < 0) {
		close(sockfd);
		if (errno == EINTR)
			errno = ETIMEDOUT;
	}
    // 关闭报警时钟，并恢复原处理函数
	alarm(0);					/* turn off the alarm */
	Signal(SIGALRM, sigfunc);	/* restore previous signal handler */

	return(n);
}

// 信号函数仅仅返回
static void connect_alarm(int signo)
{
	return;		/* just interrupt the connect() */
}
```
※ 本方法仅能减少 connect 的超时，但不能增加 connect 的超时设置，因为 connect 有自己的超时设置  
※ 需要注意的是在多线程中使用信号非常困难，建议仅仅在未线程化或仅在单线程中使用本技术  

### 2. 使用 SIGALRM 为 recvfrom 设置超时

```cpp
static void	sig_alrm(int);

void dg_cli(FILE *fp, int sockfd, const SA *pservaddr, socklen_t servlen)
{
	int	n;
	char	sendline[MAXLINE], recvline[MAXLINE + 1];

	Signal(SIGALRM, sig_alrm);

	while (Fgets(sendline, MAXLINE, fp) != NULL) {

		Sendto(sockfd, sendline, strlen(sendline), 0, pservaddr, servlen);
        // 调用 recvfrom 函数前设置了 5 秒的超时设置
		alarm(5);
		if ( (n = recvfrom(sockfd, recvline, MAXLINE, 0, NULL, NULL)) < 0) {
			if (errno == EINTR)
				fprintf(stderr, "socket timeout\n");
			else
				err_sys("recvfrom error");
		} else {
            // 读取数据，关闭超时处理
			alarm(0);
			recvline[n] = 0;	/* null terminate */
			Fputs(recvline, stdout);
		}
	}
}
// 简单返回，用来中断阻塞的 ercvfrom 调用
static void sig_alrm(int signo)
{
	return;			/* just interrupt the recvfrom() */
}
```

### 3. 使用 select 为 recvfrom 设置超时

该函数中 select 指定等待描述符的最长时间
```cpp
int readable_timeo(int fd, int sec)
{
	fd_set			rset;
	struct timeval	tv;
    // 准备 select 参数
	FD_ZERO(&rset);
	FD_SET(fd, &rset);

	tv.tv_sec = sec;
	tv.tv_usec = 0;
    // 调用有超时的 select 函数，出错返回 -1，超时返回 0
	return(select(fd+1, &rset, NULL, NULL, &tv));
		/* 4> 0 if descriptor is readable */
}
```

### 4. 使用 SO_RCVTIMEO 套接字选项为 recvfrom 设置超时

&emsp;&emsp;该操作设置一次即可，与套接字的读操作绑定，前面的方法都需要循环重新设置，本套接字选项仅适用于读操作，类似的 SO_SNDTIMEO 选项对应于写操作，两者均不能用于 connect 设置超时  
```cpp
void dg_cli(FILE *fp, int sockfd, const SA *pservaddr, socklen_t servlen)
{
	int				n;
	char			sendline[MAXLINE], recvline[MAXLINE + 1];
	struct timeval	tv;
    // 指向 timeval 结构体的指针，保存的是超时的值
	tv.tv_sec = 5;
	tv.tv_usec = 0;
	Setsockopt(sockfd, SOL_SOCKET, SO_RCVTIMEO, &tv, sizeof(tv));

	while (Fgets(sendline, MAXLINE, fp) != NULL) {

		Sendto(sockfd, sendline, strlen(sendline), 0, pservaddr, servlen);

		n = recvfrom(sockfd, recvline, MAXLINE, 0, NULL, NULL);
		if (n < 0) {
            // I/O 超时操作，recvfrom 函数返回一个 EWOULDBLOCK 错误
			if (errno == EWOULDBLOCK) {
				fprintf(stderr, "socket timeout\n");
				continue;
			} else
				err_sys("recvfrom error");
		}

		recvline[n] = 0;	/* null terminate */
		Fputs(recvline, stdout);
	}
}
```

# 14.3 recv 和 send 函数

类似于 read 和 write 函数，不过多一个参数 flags
```cpp
#include<sys/socket.h>
ssize_t recv(int sockfd, void* buff, size_t nbytes, int flags);
ssize_t send(int sockfd, const void* buff, size_t nbytes, int flags);
// 返回：成功返回读入写出的字节数，出错返回 -1
```
&emsp;&emsp;flag 可以用来标识一些操作，绕过路由表查找、仅本操作非阻塞、发送或接受外带数据、窥看外来消息、等待所有数据。具体用法参看 UNP-14.3  
&emsp;&emsp;flag 是值传递，并不是值-结果参数。所以它只能从进程向内核传递标志，内核不能返回标志。随着协议的增加，需要实现这个操作使用 recvmsg 和 sendmsg 中用 msghdr 来进行引用传递  

# 14.4 readv 和 writev 函数

&emsp;&emsp;这两个函数类似 read 和 write，不过 readv 和 writev 允许单个系统调用读入或写出自一个或者多个缓冲区。这些操作被称为分散读和集中写，因为来自读操作的输入数据被分散到多个应用缓冲区中，而来自多个应用缓冲区的输出数据被集中提供给单个写操作。  
```cpp
#include<sys/uio.h>
ssize_t readv(int fileds, const struct iovec* iov, int iovcnt);
ssize_t writev(int fields, const struct iovec* iov, int iovcnt);
// 返回：成功返回读入或写出的字节数，出错返回 -1
struct iovec{
    void *iov_base;      // buf 的开始地址
    size_t iov_len;      // buf 的大小
}
```
&emsp;&emsp;struct iovec 结构体定义如上，函数第二个参数指向的是一个 iovec 的数组，一般系统中定义数组长度的常值为 16，最大值一般在 1024~2100  
&emsp;&emsp;readv 和 writev 函数可用于任何描述符，不仅仅局限于套接字描述符。writev 是一个原子操作，所以对于 UDP 来说，一次 writev 仅产生一个 UDP 数据报  
#### 当一个 4 字节的 write 和 396 字节的 write 调用时可能会触发 naggle 算法合并它们，解决这个问题的首选方法就是使用 writev 函数  

# 14.5 recvmsg 和 sendmsg 函数

这两个函数是最通用的 I/O 函数，可以替换上面所有的读写函数  
```cpp
#include<sys/socket.h>
ssize_t recvmsg(int sockfd, struct msghdr * msg, int flags);
ssize_t sendmsg(int sockfd, struct msghdr * msg, int flags);
// 返回：成功读入或者写出的字节数，出错则为 -1

struct msghdr{   // 用来保存大部分参数
    void *mag_name;
    socklen_t msg_namelen;
    struct iovec *msg_iov;
    int msg_iovlen;
    void *msg_control;
    socklen_t msg_controllen;
    int msg_flags;
}
```
1. mag_name 和 msg_namelen 这两个成员用于套接字未连接的场合（UDP 等），msg_name 指向一个套接字地址结构，低走用着在其中存放接受者（sendmsg）或者发送者（recvmsg）的协议地址。如果无需指明协议地址，msg_name 是空指针。msg_namelen 对于 sendmsg 是值参数，对于 recvmsg 是值-结果参数。  
1. msg_iov 和 msg_iovlen 这两个成员指定输入或输出缓冲区数组（就是 iovec 数组）  
1. msg_control 和 msg_controllen 这两个成员指定可选的辅助数据的位置和大小 msg_controllen 对于 recvmsg 来说是一个值-结果参数
1. 对于 recvmsg 和 sendmsg 需要区别他们的两个标志变量，一个是传递值的 flags 参数，另一个是所传递 msghdr 结构的 msg_flags 成员，传递的是引用，因为传递给函数的是地址
- 只有 recvmsg 使用 msg_flags 参数。sendmsg 忽略该参数，相关参数设置参考 UNP-14.5  
  
### 五组 I/O 函数之间的差异
函数|任何描述符|仅套接字描述符|单个读/写缓冲区|分散/集中读/写|可选标志|可选对端地址|可选控制信息
-|-|-|-|-|-|-|-
read, write|√| |√| | | | 
readv, writev|√| | |√| | | 
recv, send| |√|√| |√| | 
recvfrom, sendto| |√|√| |√|√| 
recvmsg, sendmsg| |√| |√|√|√|√

# 14.6 辅助数据

&emsp;&emsp;辅助数据可通过调用 sendmsg 和 recvmsg 这两个函数，使用 msghdr 结构中的 msg_control 和 msg_controllen 这两个成员来发送和接收，也叫做控制信息。  
&emsp;&emsp;辅助数据由一个或多个辅助数据对象构成，每个对象以一个定义在头文件<sys/socket.h>中的 cmsghdr 结构体
```cpp
struct cmsghdr{
	socklen_t cmsg_len;
	int cmsg_level;
	int cmsg_type;
}
```

# 14.7 排队的数据量

&emsp;&emsp;有时候我们想要在不真正读取数据的前提下直到一个套接字上已有多少数据排队等待着读取。有三种技术可以获得一排队数据量。  
1. 如果获取排队数据量的目的在于避免读操作内核阻塞，呢么可以使用非阻塞式 I/O，但是不能获得数据量啊，只能直到是否有数据。
2. 如果我们既想查看数据又想数据留在接收队列中等待其余部分的读取，使用 MSG_PEEK 标志。可以使用非阻塞套接字来实现既能判断是否有数据可读，然后又可以查看数据但不读取。需要注意的是对于 TCP 连接，两次获取量的值大小可能不同，如果在两次获取之间收到了流数据。但是 UDP 仅返回第一个数据报的大小，所以即使两次之间有新的数据报，也不影响
3. 一些实现支持 ioctl 的 FIONREAD 命令。该命令的第三个 ioctl 参数是指向某个整数的一个指针，内核通过该整数返回的值就是套接字接收队列的当前字节数。该值是已排队字节的总和，对于 UDP 包括所有已排队的数据报。某些实现中，队 UDP 套接字返回的值还包括一个套接字地址结构的空间，其中含有发送者的 IP 地址和端口号

# 14.8 套接字和标准 I/O

执行 I/O 还可以使用标准 I/O 函数库，使用标准 I/O 对套接字进行读取一般可以打开两个流，一个用来读，一个用来写。  
#### 不建议在套接字上使用标准 I/O

# 14.9 高级轮询技术

### kqueue 接口
&emsp;&emsp;本接口允许进程向内核注册描述所关注的 kqueue 事件的事件过滤器。事件除了与 select 所关注类似的文件 IO 超时外，还有异步 IO、文件修改通知、进程跟踪、信号处理
```cpp
#include<sys/types.h>
#include<sys/event.h>
#include<sys/time.h>
int kqueue(void);
int kevent(int kq, const struct kevent * changelist, int nchanges,
			struct kevent * eventlist, int nevents,
			const struct timespec * timeout);
void EV_SET(struct kevent *kev, uintptr_t ident, short filter, 
			u_short flags, u_int fflags, intptr_t data, void *udata);

struct kevent{
	uintptr_t ident;
	short filter;
	u_short flags;
	u_int fflags;
	intptr_t data;
	void *udata;
}
```

&emsp;&emsp;kqueue 函数返回一个新的 kqueue 描述符，用于后面的 kevent 调用。kevent 函数即用于注册所关注的事件，也用于确定是否有所关注事件发生。changelist 和 nchanges 这两个参数给出对所关注事件做出更改，没有的话设置为 NULL，0。关于 timeout 结构体的区别，select 是纳秒，而 kqueue 是微秒。

# 14.10 t/tcp：事务目的 tcp

&emsp;&emsp;T/TCP 是对 TCP 的略微修改，避免最近通信过的主机之间再次三次握手。它能把 SYN，FIN 和数据组合到单个分节中，前提是一个分节可以存储这些数据。  
&emsp;&emsp;T/TCP 包含所有 TCP 的特性，使得基于 TCP 的连接有了类似于 UDP 的效果，即两个主机之间频繁连接断开，但是使用 T/TCP 可以使得三次握手的消耗几乎为 0  

# 小结

1. 在套接字操作上设置时间限制的方法有三个：
- 使用 alarm 函数和 SIGALRM 信号。
- 使用由 select 提供的时间限制
- 使用较新的 SO_RCVTIMEO 和 SO_SNDTIMEO 套接字选项  

2. 第一种方法简单易用，但是涉及信号处理，可能引发竞争条件。使用 select 会阻塞在 select 上，而不是阻塞在 read，write，connect 调用上。第三种方法不是所有系统都提供。  
3. recvmsg 和 sendmsg 是 5 组读写函数中最通用的。它有其余读写函数的所有特性：指定 MSG_xxx，返回或指定对端的协议地址，使用多个缓冲区，还增加了两个新特性：给应用进程返回标志，接收或者发送辅助数据。  
4. C 标准 I/O 可以用在套接字上，但是并不推荐使用。  
5. T/TCP 是对 TCP 的简单增强版本，能够在两台主机最近通信的前提下避免三路握手，使得对端更快的做出应答。从编程角度看，客户端通过调用 sendto 而不是通常使用的 connect write shutdown 调用序列发挥 T/TCP 的优势