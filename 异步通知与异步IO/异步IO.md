# Linux异步I/O
## AIO概念与GNU C库AIO
Linux中最常用的I/O模型是同步I/O。在这个模型中，当请求发出之后，应用程序就会阻塞，直到请求满足为止。这是一种很好的解决方案，调用应用程序在等待I/O请求完成时不需要占用CPU。但在许多场景中，I/O请求可能需要与CPU消耗产生交叠，已充分利用CPU和I/O提高吞吐率。

异步I/O在应用程序发起I/O动作之后，直接开始执行，并不等待I/O结束，它要么过一段时间来查询之前的I/O完成情况，要么I/O完成之后自动被调用与I/O完成绑定的回调函数。

### glibc库的AIO
Linux的AIO有多种实现，其中一种实现是在用户空间的glibc库中实现的，它本质上是借用了多线程模型，用开启新的线程以同步的方法来做I/O，新的AIO辅助线程与发起AIO的线程以pthread_cond_signal()的形式进程线程间的同步。glibc的AIO主要包括如下函数：
>1. aio_read()  
aio_read()函数请求对一个有效的文件描述符进行异步读操作。这个文件描述符可以是一个文件、套接字、甚至是管道。aio_read()的函数原型如下：
```c
int aio_read(struct aiocb *aiocbp);
```
aio_read()函数在请求进行排队之后会立即返回（尽管读操作并未完成）。如果执行成功，返回值就为0，如果出现错误，返回值就为-1，并设置errno的值。

参数aiocb(AIO I/O Control Block)结构体包含了传输的所有信息，以及为AIO操作准备的用户空间缓冲区。在产生I/O完成通知时，aiocb结构就被用来唯一标识所完成的I/O操作。

>2. aio_write()  
aio_write()函数用来请求一个异步写操作，原型如下：
```c
int aio_write(struct aiocb *aiocbp);
```
aio_write()会立即返回，并且它的请求已经被排队（成功时返回0，失败时返回-1，并设置相应errno）。

>3. aio_error()  
aio_error()函数被用来确定请求的状态。原型如下：
```c
int aio_error(struct aiocb *aiocbp);
```
这个函数可以返回如下内容：  
EINPROGRESS：说明请求尚未完成  
ECANCELED：说明请求被应用程序取消了
-1：说明发生了错误，具体错误原因由errno记录

>4. aio_return()  
异步I/O和同步阻塞I/O方式之间的一个区别是不能立即访问这个函数的返回状态，因为异步I/O并没有阻塞在read()调用上。在标准的同步阻塞read()调用中，返回状态是在该函数返回时提供的。但是在异步I/O中，我们要用aio_return()函数，其原型如下：
```c
ssize_t aio_return(struct aiocb *aiocbp);
```
只有在aio_error()调用确定请求已经完成（可能成功，也可能发生了错误）之后，才会调用这个函数。aio_return()的返回值就等价于同步情况的read()或write()系统调用的返回值（所传输的字节数，如果发生错误，返回值为负数）。

下面代码给出了用户空间应用程序进行异步读操作的一个例程，它首先打开文件，然后准备aiocb结构体，之后调用aio_read(&my_aiocb)进行提出异步读请求，当aio_error(&my_aiocb) == EINPROGRESS，即操作还在进行中时，一直等待，结束后通过aio_return(&my_aiocb)获得返回值。

```c
#include <aio.h>
...
int fd, ret;
struct aiocb my_aiocb;

fd = open("file.txt", O_RDONLY);
if (fd < 0)
	perror("open");

/* 清零aiocb结构体 */
bzero(&my_aiocb, sizeof(struct aiocb));

/* 为aiocb请求分配数据缓冲区 */
my_aiocb.aio_buf = malloc(BUFSIZE + 1);
if (!my_aiocb.aio_buf)
	perror("malloc");

/* 初始化aiocb成员 */
my_aiocb.aio_fildes = fd;
my_aiocb.aio_nbytes = BUFSIZE;
my_aiocb.aio_offset = 0;

ret = aio_read(&my_aiocb);
if (ret < 0)
	perror("aio_read");

while (aio_error(&my_aiocb) == EINPROGRESS)
	continue;

if ((ret = aio_return(&my_aiocb)) > 0) {
	/* 获得异步读的返回值 */
} else {
	/* 读失败，分析errorno */
}
```

>5. aio_suspend()  
用户可以使用aio_suspend()函数来阻塞调用进程，直到异步请求完成为止。调用者提供了一个aiocb引用列表，其中任何一个完成都会导致aio_suspend()返回。aio_suspend()的原型如下：
```c
int aio_suspend(const struct aiocb *const cblist[],
						int n, const struct timespec *timeout);
```

如下代码给出了用户空间进行异步读操作时使用aio_suspend()函数的例子：
```c
struct aiocb *cblist[MAX_LIST];
/* 清零aiocb结构体链表 */
bzero((char *)cblist, sizeof(cblist));
/* 将一个或更多的aiocb放入aiocb结构体链表中 */
cblist[0] = &dmy_aiocb;
ret = aio_read(&my_aiocb);
ret = aio_suspend(cblist, MAX_LIST, NULL);
```

当然，在glibc实现的AIO中，除了上述同步的等待方式之外，也可以使用信号或者回调机制来异步地标明AIO的完成。

>6. aio_cancel()  
aio_cancel()函数允许用户取消对一个文件描述符进行的一个或所有I/O请求，其原型如下：
```c
int aio_cancel(int fd, struct aiocb *aiocbp);
```
要取消一个请求，用户需提供文件描述符和aiocb指针。如果这个请求被成功取消了，那么这个函数返回AIO_CANCELED，如果请求完成了，返回AIO_NOTCANCELED。

要取消对某个给定文件描述符的所有请求，用户需要提供这个文件的描述符，并将aiocbp参数设为NULL。如果所有的请求都取消了，这个函数就会返回AIO_CANCELED；如果至少有一个请求没被取消，那么就返回AIO_NOT_CANCELED；如果没有一个请求可以被取消，那么就返回AIO_ALLDONE。然后，可以使用aio_error()来验证每个AIO请求，如果某请求已经被取消了，那么aio_error()就会返回-1，并且errno会被设置为ECANCELED

>7. lio_listio()  
lio_listio()函数可用于同时发起多个传输。这个函数非常重要，它使得用户可以在一个系统调用中启动大量的I/O操作。其原型如下：
```c
int lio_listio(int mode, struct aiocb *list[], int nent, struct sigevent *sig);
```
mode参数可以时LIO_WAIT或LIO_NOWAIT。LIO_WAIT会阻塞这个调用，直到所有的I/O都完成为止。但是若是LIO_NOWAIT模型，在I/O操作进行排队之后，该函数就会返回。list是一个aiocb引用的列表，最大元素的个数有nent决定的。如果list的元素为NULL，lio_listio()会将其忽略。

下面的代码给出了用户空间进行异步I/O操作时使用lio_listio()函数的例子。
```c
struct aiocb aiocb1, aiocb2;
struct aiocb *list[MAX_LIST];
...
/* 准备第一个aiocb */
aiocb1.aio_fildes = fd;
aiocb1.aio_buf = malloc(BUFSIZE + 1);
aiocb1.aio_nbytes = BUFSIZE;
aiocb1.aio_offset = next_offset;
aiocb1.aio_lio_opcode = LIO_READ;			/* 异步读操作 */
...		/* 准备多个aiocb */
bzero((char *)list, sizeof(list));

/* 将aiocb填入链表 */
list[0] = &aiocb1;
list[1] = &aiocb2;
...
ret = lio_listio(LIO_WAIT, list, MAX_LIST, NULL);	/* 发起大量I/O操作 */
```
上述代码，因为是进行异步读操作，所以操作码是LIO_READ，而对于写操作，操作码为LIO_WRITE，而LIO_NOP是空操作。

网页http://www.gnu.org/software/libc/manual/html_node/Asynchronous-I_002fO.html包含了AIO库函数的详细信息。


### Linux内核AIO与libaio
Linux AIO也可以由内核空间实现，异步I/O是Linux 2.6版本以后内核的一个标准特性。对于块设备而言，AIO可以一次性发出大量的read/write调用并且通过通用块层的I/O调度来获得更好的性能，用户程序也可以减少过多的同步负载，还可以在业务逻辑中更灵活地进行并发控制和负载均衡。相较于glibc的用户空间多线程同步等实现也减少了线程的负载和上下文切换等。对于网络设备而言，在socket层面上，也可以使用AIO，让CPU和网卡的收发动作充分交叠以改善吞吐性能。选择正确的I/O模型对系统的性能影响很大，这部分可以参阅C10K问题，详见网址http://www.kegel.com/c10k.html。

在用户空间中，我们一般要结合libaio来进行内核AIO的系统调用。内核AIO提供的系统调用主要包括：
```c
int io_setup(int maxevents, io_context_t *ctxp);
int io_destroy(io_context_t ctx);
int io_submit(io_context_t ctx, long nr, struct iocb *ios[]);
int io_cancel(io_context_t ctx, struct iocb *iocb, struct io_event *evt);
int io_getevents(io_context_t ctx_id, long min_nr, long nr, struct io_event *events,
	struct timespec *timeout);
void io_set_callback(struct iocb *iocb, io_callback_t cb);
void io_prep_pwrite(struct iocb *iocb, int fd, void *buf, size_t count, long long offset);
void io_prep_pread(struct iocb *iocb, int fd, void *buf, size_t count, long long offset);
void io_prep_pwritev(struct iocb *iocb, int fd, const struct iovec *iov, int iovcnt, long long offset);
void io_prep_preadv(struct iocb *iocb, int fd, const struct iovec *iov, int iovcnt, long long offset);
```

AIO的读写请求都用io_submit()下发。下发前通过io_prep_pwrite()和io_prep_pread()生成iocb的结构体，作为io_submit()的参数。这个结构体指定了读写类型、起始地址、长度和设备标志符等信息。读写请求下发之后，使用io_getevents()函数等待I/O完成事件。io_set_callback()则可设置一个AIO完成的回调函数。

下面代码演示了一个简明的利用libaio向内核发起AIO请求的模板。该程序编译使用命令gcc aior.c -o aior -laio编译，运行时带一个文本文件路径作为参数，该程序会打印这个文件前4096个字节的内容：
```c
#define _GNU_SOURCE			/* O_DIRECT is not POSIX */
#include <stdio.h>			/* for perror() */
#include <unistd.h>			/* for syscall() */
#include <fcntl.h>			/* O_RDWR */
#include <string.h>			/* memset() */
#include <inttypes.h>		/* uint64_t */
#include <stdlib.h>

#include <libaio.h>

#define BUF_SIZE	4096

int main(int argc, char **argv)
{
	io_context_t ctx = 0;
	struct iocb cb;
	struct iocb *cbs[1];
	unsigned char *buf;
	struct io_event events[1];
	int ret;
	int fd;

	if (argc < 2) {
		printf("the command format: aior [FILE]\n");
		exit(1);
	}

	fd = open(argv[1], O_RDWR | O_DIRECR);
	if (fd < 0) {
		perror("open error");
		goto err;
	}

	/* Allocate aligned memory */
	ret = posix_memalign((void **)&buf, 512, (BUF_SIZE + 1));
	if (ret < 0) {
		perror("posix_memalign failed");
		goto err1;
	}
	memset(buf, 0, BUF_SIZE + 1);

	ret = io_setup(128, &ctx);
	if (ret < 0) {
		printf("io_setup error:%s", strerror(-ret));
		goto err2;
	}

	/* setup I/O control block */
	io_prep_pread(&cb, fd, buf, BUF_SIZE, 0);

	cbs[0] = &cb;
	ret = io_submit(ctx, 1, cbs);
	if (ret != 1) {
		if (ret < 0)
			printf("io_submit error:%s", strerror(-ret));
		else
			fprintf(stderr, "could not submit IOs");
		goto err3;
	}

	/* get the reply */
	ret = io_getevents(ctx, 1, 1, events, NULL);
	if (ret != 1) {
		if (ret < 0)
			printf("io_getevents error:%s", strerror(-ret));
		else
			fprintf(stderr, "could not get Events");
		goto err3;
	}
	if (events[0].res2 == 0) {
		printf("%s\n", buf);
	} else {
		printf("AIO error:%s", strerror(-events[0].res));
		goto err3;
	}

	if ((ret = io_destroy(ctx)) < 0) {
		printf("io_destroy error:%s", strerror(-ret));
		goto err2;
	}

	free(buf);
	close(fd);
	return 0;

err3:
	if ((ret = io_destroy(ctx)) < 0)
		printf("io_destroy error:%s", strerror(-ret));

err2:
	free(buf);

err1:
	close(fd);

err:
	return -1;
}
```