# Linux中断编程

## 申请和释放中断
在Linux设备驱动中，使用中断的设备需要申请和释放对应的中断，分别使用内核提供的request_irq()和free_irq()函数。

### 1. 申请irq
```c
int request_irq(unsigned int irq, irq_handler_t handler, unsigned long irqflags, const char *name, void *dev);
```
irq是要申请的硬件中断号。  
handler是向系统登记的中断处理函数（顶半部），是一个回调函数，中断发生时，系统调用这个函数，dev参数将被传递给它。  
irqflags是中断处理的属性，可以指定中断的触发方式和处理方式。在触发方式方面，可以是IRQF_TRIGGER_RISING、IRQF_TRIGGER_FALLING、IRQF_TRIGGER_HIGH、IRQF_TRIGGER_LOW等。在处理方式方面，若设置了IRQF_SHARED，则表示多个设备共享中断，dev是要传递给中断处理程序的私有数据，一般设置为这个设备的设备结构体或者NULL。  
request_irq()返回0表示成功，返回-EINVAL表示中断号无效或处理函数指针为NULL，返回-EBUSY表示中断被占用且不能共享。

```c
int devm_request_irq(struct device *dev, unsigned int irq, irq_handler_t handler, unsigned long irqflags, const char *devname, void *dev_id);
``
此函数与request_irq()的区别是devm_开头的API申请的是内核“managed”的资源，一般不需要再出错处理和remove()接口里再显示释放。有点类似java的垃圾回收机制。

顶半部handler的类型irq_handler_t定义为：
```c
typedef irqreturn_t (*irq_handler_t)(int, void *);
typedef int irqreturn_t;
```

### 2. 释放irq
与request_irq()相对应的函数free_irq()的原型：
```c
void free_irq(unsigned int irq, void *dev_id);
```
参数与request_irq()相同。

## 使能和屏蔽中断
使能一个中断：
```c
void enable_irq(int irq);
```

下列两个函数用于屏蔽一个中断源：
```c
void disable_irq(int irq);
void disable_irq_nosync(int irq);
```
两者的区别在于disable_irq_nosync()立即返回，而disable_irq()等待目前的中断处理完成。由于disable_irq()等待目前的中断处理完成，因此如果在n号中断的顶半部调用disable_irq(n)会引起系统死锁，这种情况下，只能调用disable_irq_nosync(n)。

下列两个函数（或宏，具体实现依赖CPU体系结构）将屏蔽本CPU内所有中断：
```c
#define local_irq_save(flags) ...
void local_irq_disable(void);
```
前者会将目前的中断状态保留在flags中（注意flags为unsigned long类型，被直接传递，而不是通过指针），后者直接禁止中断而不保存状态。

与上述两个禁止中断对应的恢复中断的函数（或宏）：
```c
#define local_irq_restore(flags) ...
void local_irq_enable(void);
```
以上各local_开头的方法作用范围是本CPU内。