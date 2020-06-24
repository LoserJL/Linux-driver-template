# 阻塞与非阻塞I/O
阻塞操作是指，在执行设备操作时，若不能获得资源，则挂起进程直到满足可操作的条件后再进行操作。被挂起的进程进入睡眠状态，被从调度器的运行队列移走，直到等待的条件被满足。而非阻塞操作的进程在不能进行设备操作时，并不挂起，它要么放弃，要么不断查询，直至可以进行操作为止。

驱动程序通常需要提供这样的能力：当应用程序执行read()、write()等系统调用时，若设备的资源不能获取，而用户又希望以阻塞的方式访问设备，驱动程序应在设备驱动的xxx_read()、xxx_write()等操作中将进程阻塞知道资源可以获取，此后，应用程序的read()、write()等调用才返回，整个过程仍然进行了正确的设备访问，用户并没有感知到；若用户以非阻塞的方式访问设备文件，则当设备资源不可获取时，设备驱动的xxx_read()、xxx_write()等操作应立即返回，read()、write()等系统调用也随即被返回，应用程序收到-EAGAIN返回值。

在阻塞访问时，不能获取资源的进程将进入休眠，它将CPU资源“礼让”给其他进程。因为阻塞的进程会进入休眠，所以必须确保有一个地方能够唤醒休眠的进程，否则，进程就真的“寿终正寝”了。唤醒进程的地方最大可能发生在中断里，因为在硬件资源获得的同时往往伴随着一个中断。而非阻塞的进程不断尝试，直至可以进行I/O。

下面两段代码分别演示了以阻塞和非阻塞方式读取串口一个字符的代码。前者在打开文件时没有O_NONBLOCK标记，后者使用O_NONBLOCK标记打开文件。

**阻塞地读串口一个字符**
```c
char buf;
fd = open("/dev/ttyS1", O_RDWR);
...
res = read(fd, &buf, 1);	/* 当串口上有输入时才返回 */
if (res == 1)
	printf("%c\n", buf);
```

**非阻塞地读串口一个字符**
```c
char buf;
fd = open("/dev/ttyS1", O_RDWR | O_NONBLOCK);
...
while(read(fd, &buf, 1) != 1)
	continue;		/* 串口上无输入也返回，因此要不断尝试 */
printf("%c\n", buf);
```

除了在打开文件时可以指定是阻塞还是非阻塞方式以外，在文件打开后，也可以通过ioctl()和fcntl()改变读写的方式，如从阻塞变更为非阻塞或者从非阻塞变更为阻塞。例如：
```c
fcntl(fd, F_SETFL, O_NONBLOCK);
```
可以设置fd对应的I/O为非阻塞。

## 等待队列
在Linux驱动程序中，可以使用等待队列（Wait Queue）来实现阻塞进程的唤醒。

Linux内核提供了如下关于等待队列的操作：
### 1. 定义等待队列头部
```c
wait_queue_head_t my_queue;
```
wait_queue_head_t是__wait_queue_head结构体的一个typedef。

### 2. 初始化"等待队列头部"
```c
init_waitqueue_head(&my_queue);
```
**而下面的DECLARE_WAIT_QUEUE_HEAD()宏可以作为定义并初始化等待队列头部的"快捷方式"：**
```c
DECLARE_WAIT_QUEUE_HEAD(name)
```

### 3. 定义等待队列元素
```c
DECLARE_WAITQUEUE(name, tsk)
```
该宏用于定义并初始化一个名为name的等待队列元素。

### 4. 添加/移除等待队列
```c
void add_wait_queue(wait_queue_head_t *q, wait_queue_t *wait);
void remove_wait_queue(wait_queue_head_t *q, wait_queue_t *wait);
```
add_wait_queue()用于将等待队列元素wait添加到等待队列头部q指向的双向链表中，而remove_wait_queue()用于将等待队列元素wait从由q头部指向的双向链表中移除。

### 5. 等待事件
```c
wait_event(queue, condition)
wait_event_interruptible(queue, condition)
wait_event_timeout(queue, condition, timeout)
wait_event_interrruptible_timeout(queue, condition, timeout)
```
等待第一个参数queue作为等待队列头部的队列被唤醒，而且第二个参数condition必须满足，否则继续阻塞。wait_event()和wait_event_interruptible()的区别是后者能被信号打断，而前者不能。加上timeout参数的宏意味着阻塞等待的超时时间，以jiffy为单位，在第三个参数timeout到达时，不论condition是否满足，均返回。

### 6. 唤醒队列
```c
void wake_up(wait_queue_head_t *queue);
void wake_up_interruptible(wait_queue_head_t *queue);
```
上述操作会唤醒以queue作为等待队列头部的队列中所有的进程。

wake_up()应与wait_event()或wait_event_timeout()成对使用，而wake_up_interruptible()则应与wait_event_interruptible()或wait_event_interruptible_timeout()成对使用。wake_up()可唤醒处于TASK_INTERRUPTIBLE和TASK_UNINTERRUPTIBLETASK_UNINTERRUPTIBLE的进程，而wake_up_interruptible()只能唤醒处于TASK_INTERRUPTIBLE的进程。

### 7. 在等待队列上睡眠
```c
sleep_on(wait_queue_head_t *q);
interruptible_sleep_on(wait_queue_head_t *q);
```
sleep_on()函数的作用就是将目前进程的状态置成TASK_UNINTERRUPTIBLE，并定义一个等待队列元素，之后把它挂到等待队列头部q指向的双向链表，直到资源可获得，q队列指向链接的进程被唤醒。  
interruptible_sleep_on()与sleep_on()类似，其作用是将目前进程的状态置成TASK_INTERRUPTIBLE，并定义一个等待队列元素，之后把它附属到q指向的队列，直到资源可获得（q指引的等待队列被唤醒）或者进程收到信号。

sleep_on()应该与wake_up()成对使用，interruptible_sleep_on()应该与wake_up_interruptible()成对使用。

下面的代码演示了一个在驱动程序中使用等待队列的模板，在进行写I/O操作的时候，判断设备是否可写，如果不可写且为阻塞I/O，则进程睡眠并挂起到等待队列。
```c
static ssize_t xxx_write(struct file *file, const char *buffer, size_t count, loff_t *ppos)
{
	...

	DECLARE_WAITQUEUE(wait, current); /* 定义等待队列元素 */
	add_wait_queue(&xxx_wait, &wait); /* 添加元素到等待对列 */

	/* 等待设备缓冲区可写 */
	do {
		avail = device_writable(...);
		if (avail < 0) {
			if (file->f_flags & O_NONBLOCK) { //非阻塞
				ret = -EAGAIN;
				goto out;
			}
			__set_current_state(TASK_INTERRUPTIBLE); /* 改变进程状态 */
			schedule(); /* 调度其他进程执行 */
			
			if (signal_pending(current)) { //如果是因为信号唤醒
				ret = -ERESTARTSYS;
				goto out;
			}
		}
	} while (avail < 0);

	/* 写设备缓冲区 */
	device_write(...);

	out:
	remove_wait_queue(&xxx_wait, &wait); /* 将元素移除xxx_wait指引的队列 */
	set_current_state(TASK_RUNNING); /* 设置进程状态为TASK_RUNNING */
	return ret;
}
```
读懂以上代码对理解Linux进程状态切换非常重要，几个要点如下：
>1. 如果是非阻塞访问（O_NONBLOCK被设置），设备忙时，直接返回-EAGAIN  
>2. 对于阻塞访问，会调用__set_current_state(TASK_INTERRUPTIBLE)进行进程状态切换并显示通过schedule()调度其他进程执行  
>3. 醒来的时候要注意，由于调度出去的时候，进程状态是TASK_INTERRUPTIBLE，即浅度睡眠，所以唤醒它的有可能是信号，因此，我们首先通过signal_pending(current)了解是不是信号唤醒的，如果是，立即返回-ERESTARTSYS  

DECLARE_WAITQUEUE、add_wait_queue这两个动作加起来完成的效果：在wait_queue_head_t指向的链表上(add_wait_queue的第一个参数)，新定义的wait_queue元素(add_wait_queue的第二个参数，也是DECLARE_WAITQUEUE的第一个参数)被插入，而这个新插入的元素绑定了一个task_struct（当前做xxx_write的current，这也是DECLARE_WAITQUEUE使用“current”作为参数的原因）