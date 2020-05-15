# Linux字符驱动中私有数据使用方法

### 一、前提

为了代码更加整洁， 尽量少用全局变量，linux引入私有数据概念， 通过学习别人代码和自己理解，写字符驱动首先创建一个描述当前字符驱动的结构体，只是描述字符驱动不与业务耦合，再创建一个与业务相关的驱动上下文结构体。把驱动上下文结构体当做私有数据保存到驱动框架里。

```c
struct _chr_env
{
	int major;           /* 该字符驱动主设备号，申请得到 */
	int number;          /* 有多少个次设备 */
	struct class *class; /* 该字符驱动的class */
};

// 驱动上下文结构体
typedef struct _key_irq_dev
{
	struct cdev cdev;    /* 该字符驱动设备描述，这个很重要，open函数会通过cdev找到该结构体首地址*/
	dev_t devid;         // 以下变量根据自己业务添加
......
} key_irq_dev;

static struct _chr_env chr_env; /* 全局就这一个全局变量，这个省不了，init exit函数里要用到 */
```

### 二、在probe函数里申请资源, 然后保存到platform_device

```c
static int gpio_key_probe(struct platform_device *dev)
{
    struct device_node *node = dev->dev.of_node;
	int count = of_gpio_count(node);   
    key_irq_dev *key_ctx = kmalloc(sizeof(key_irq_dev) * count, GFP_KERNEL);
	platform_set_drvdata(dev, key_ctx);
    .....
}
```

### 三、remove函数里拿到设备上下文结构体

```c
static int gpio_key_remove(struct platform_device *dev)
{
	key_irq_dev *key_ctx = (key_irq_dev *)platform_get_drvdata(dev);
    ......
    kfree(key_ctx);
}
```

### 四、Linux 字符驱动open函数中使用私有数据

```c
static int key_open (struct inode *node, struct file *file)
{
	key_irq_dev *key_ctx = container_of(node->i_cdev, key_irq_dev, cdev);
	
	file->private_data = key_ctx;
	
	return 0;
}
```

Linux 驱动中`container_of()`函数可谓四两拨千斤，通过结构体中一个变量找到这个结构体的首地址，从而拿到之前申请的驱动上下文结构体，在之前probe里申请了count个key_irq_dev结构体，每个驱动节点对应一个key_irq_dev结构体的上下文。`container_of()`可以区分是哪个节点上下文吗？

**可以**

`key_irq_dev *key_ctx = kmalloc(sizeof(key_irq_dev) * 4, GFP_KERNEL);`

| 字符设备1        | 字符设备2        | 字符设备3       | 字符设备4        |
| :--------------- | :--------------- | :-------------- | :--------------- |
| cdev,devid...... | cdev,devid...... | cdev,devid..... | cdev,devid...... |

假如申请了4个字符设备上下文，每个里面都有cdev成员，都会在`cdev_add()`时候添加到`struct file`里，在open时候取出来把单个字符设备首地址再放到`file->private_data`里，从而`close、write、read、ioctrl`都可以直接拿到。

```c
static int key_close (struct inode *node, struct file *file)
{
	key_irq_dev *key_ctx = (key_irq_dev *)file->private_data;

	return 0;
}
static ssize_t key_read (struct file *file, char __user *buf, size_t size, loff_t *offset)
{
	key_irq_dev *key_ctx = (key_irq_dev *)file->private_data;

	return 0;
}
static ssize_t key_write (struct file *file, const char __user *buf, size_t size, loff_t *offset)
{
	key_irq_dev *key_ctx = (key_irq_dev *)file->private_data;
	
	return 0;
}
```

### 