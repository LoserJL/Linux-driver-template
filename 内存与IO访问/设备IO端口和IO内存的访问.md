# 设备I/O端口和I/O内存的访问
设备通常会提供一组寄存器来控制设备、读写设备和获取设备状态，即控制寄存器、数据寄存器和状态寄存器，这些寄存器可能位于I/O空间中，也可能位于内存空间中，当位于I/O空间中，通常被称为I/O端口，当位于内存空间中，对应的内存空间被称为I/O内存。

## Linux I/O端口和I/O内存访问接口
（由于在ARM中，一般都是I/O内存，所以这里不讲I/O端口了）

### I/O内存
在内核中访问I/O内存（通常时芯片内部的各个i2c、spi、usb等控制器的寄存器或外部内存总线上的设备）之前，需要先使用ioremap()函数将设备所处的物理地址映射到虚拟地址上。ioremap()原型如下：
```c
void *ioremap(unsigned long offset, unsigned long size);
```
ioremap()与vmalloc()类似，也需要建立新的页表，但是它并不进行vmalloc()中所执行的内存分配行为。ioremap()返回一个特殊的虚拟地址，该地址可用来存取特定的物理地址范围，这个虚拟地址位于vmalloc()映射区域。通过ioremap()获得的虚拟地址应该使用iounmap()释放：
```c
void iounmap(void *addr);
```

ioremap()有个变体是devm_ioremap()，类似于其它以devm_开头的函数，通过devm_ioremap()进行的映射通常不需要再驱动退出或出错时调用iounmap()。devm_ioremap()原型：
```c
void __iomem devm_ioremap(struct device *dev, resource_size_t offset,
						unsigned long size);
```
在设备的物理地址（一般都是寄存器）被映射到虚拟地址之后，尽管可以直接通过指针访问这些地址，但是内核推荐用一组标准的API来完成相关操作。

读寄存器用readb_relaxed()、readw_relaxed()、readl_relaxed()、readb()、readw()、readl()这一组API，以分别读8bit、16bit、32bit的寄存器，没有_relaxed后缀的版本和有_relaxed后缀的版本的区别是没有后缀的包含一个内存屏障，如：
```c
#define readb(c)		({u8 __v = readb_relaxed(c); __iormb(); __v;})
#define readw(c)		({u16 __v = readb_relaxed(c); __iormb(); __v;})
#define readl(c)		({u32 __v = readb_relaxed(c); __iormb(); __v;})
```

写寄存器用writeb_relaxed()、writew_relaxed()、writel_relaxed()、writeb()、writew()、writel()这一组API，以分别写8bit、16bit、32bit的寄存器，没有_relaxed后缀的版本与有_relaxed后缀的版本的区别是前者包含一个内存屏障，如：
```c
#define writeb(v, c)		({__iowmb(); writeb_relaxed(v, c);})
#define writew(v, c)		({__iowmb(); writew_relaxed(v, c);})
#define writel(v, c)		({__iowmb(); writel_relaxed(v, c);})
```

## 申请与释放设备的I/O端口和I/O内存
（由于在ARM中，一般都是I/O内存，所以这里不讲I/O端口了）
### I/O内存申请
Linux内核提供了一组函数用于申请和释放I/O内存的范围。此处的“申请”表明该驱动要访问这片区域，它不会做任何内存映射的动作，更多的是类似于“reservation”的概念。
```c
struct resource *request_mem_region(unsigned long start, unsigned long len, char *name);
```
这个函数向内核申请n个内存地址，这些地址从first开始，name参数为设备的名称。如果分配成功，则返回值不是NULL，如果返回NULL，则意味着申请I/O内存失败。

当用request_mem_region()申请的I/O内存使用完成后，应当用release_mem_region()函数将它们还给系统，这个函数原型如下：
```c
void release_mem_region(unsigned long start, unsigned long len);
```
request_mem_region()也有变体，为devm_request_mem_region()。


## 设备I/O端口和I/O内存访问流程
（由于在ARM中，一般都是I/O内存，所以这里不讲I/O端口了）
I/O访问步骤：首先调用request_mem_region()申请资源，接着将寄存器地址通过ioremap()映射到内核空间虚拟地址，之后就可以通过Linux设备访问编程接口（如readb、readl、writeb、writel等）访问这些设备的寄存器了，访问完成后，调用iounmap()释放ioremap()申请的虚拟地址，并调用release_mem_region()释放request_mem_region()申请的I/O内存资源。

有时候，驱动在访问寄存器之前，会省去request_mem_region()这样的调用。

## 将设备地址映射到用户空间
### 1. 内存映射与VMA
一般情况下，用户空间是不可能也不应该直接访问设备的，但是设备驱动程序中可实现mmap()函数，这个函数能够使用户空间可以直接访问设备的物理地址。实际上，mmap()实现了这样一个映射过程：它将用户空间的一段内存与设备内存关联，当用户访问用户空间的这段地址范围时，实际上会转化为对设备的访问。

这种能力对于显示适配器一类的设备非常有意义，如果用户空间可以直接通过内存映射访问显存的话，屏幕帧的各点像素将不再需要一个从用户空间到内核空间复制的过程。

mmap()必须以PAGE_SIZE为单位进行映射，实际上，内存只能以页为单位进行映射，若要映射非PAGE_SIZE整数倍的地址范围，要先进行页对齐，强行以PAGE_SIZE的数倍大小进行映射。

从file_operations文件操作结构体可以看出，驱动中mmap()函数原型如下：
```c
int (*mmap)(struct file *, struct vm_area_struct *);
```
驱动中mmap()函数将在用户进行mmap()系统调用时最终被调用，mmap()系统调用的函数原型与驱动中的mmap()原型区别很大，如下所示：
```c
caddr_t mmap(caddr_t addr, size_t len, int prot, int flags, int fd, off_t offset);
```
参数fd为文件描述符，一般由open()返回，fd也可以指定为-1，此时需要指定flags参数中的MAP_ANON，表明进行的是匿名映射。

len是映射到调用用户空间的字节数，它从被映射文件开头offset个字节开始算起，offset一般为0，表示从文件头开始映射。

prot参数指定访问权限，可取如下几个值的“或”：PROT_READ(可读)、PROT_WRITE(可写)、PROT_EXEC(可执行)和PROT_NONE(不可访问)。

参数addr指定文件应被映射到用户空间的起始地址，一般被指定为NULL，这样，选择起始地址的任务就交给内核完成，而函数的返回值就是映射到用户空间的地址，其类型caddr_t实际上就是void *。

当用户调用mmap()的时候，内核会进行如下处理：
> 1. 在进程的虚拟空间查找一块VMA。  
> 2. 将这块VMA进程映射。  
> 3. 如果驱动程序或文件系统的file_operations定义了mmap()，则调用它。  
> 4. 将这个VMA插入进程的VMA链表。  

file_operations中mmap()的第一个参数就是步骤1找到的VMA。 

由mmap()系统调用映射的内存可由munmap()接触映射，这个函数的原型如下：
```c
int munmap(caddr_t addr, size_t len);
```

驱动程序中mmap()的机制是建立页表，并填充VMA结构体中vm_operations_struct指针。VMA就是vm_area_struct，用于描述一个虚拟内存区域，VMA结构体的定义如下：
```c
struct vm_area_struct {
	/* The first cache line has the info for VMA tree walking. */

	unsigned long vm_start;			/* Our start address within vm_mm. */
	unsigned long vm_end;			/* The first byte after our end address
									within vm_mm. */
	
	/* linked list of VM areas per task, sorted by address */
	struct vm_area_struct *vm_next, *vm_prev;

	struct rb_node vm_rb;

	...

	/* Second cache line starts here. */

	struct mm_struct *vm_mm;		/* The address space we belong to. */
	pgprot_t vm_page_prot;			/* Access permissions of this VMA. */
	unsigned long vm_flags;			/* Flags, see mm.h */

	...
	const struct vm_operations_struct *vm_ops;

	/* Information about our backing store: */
	unsigned long vm_pgoff;			/* Offset (within vm_file) in PAGE_SIZE
									units, *not* PAGE_CACHE_SIZE */
	struct file * vm_file;			/* File we map to (can be NULL). */
	void * vm_private_data;			/* was vm_pte (shared mem) */
	...
};
```

VMA结构体描述的虚拟地址介于vm_start和vm_end之间，而其vm_ops成员指向这个VMA的操作集。针对VMA的操作都被包含在vm_operations_struct结构体中，vm_operations_struct结构体定义如下：
```c
struct vm_operations_struct {
	void (*open)(struct vm_area_struct *area);
	void (*close)(struct vm_area_struct *area);
	int (*fault)(struct vm_area_struct *vma, struct vm_fault *vmf);
	void (*map_pages)(struct vm_area_struct *vma, struct vm_fault *vmf);

	/* notification that a previously read-only page is about to become
	 * writable, if an error is returned it will cause a SIGUBS */
	int (*page_mkwrite)(struct vm_area_struct *vma, struct vm_fault *vmf);

	/* called by access_process_vm when get_user_pages() fails, typically
	 * for use by special VMAs that can switch between memory and hardware
	 */
	int (*access)(struct vm_area_struct *vma, unsigned long addr,
			void *buf, int len, int write);
	...
};
```
整个vm_operations_struct结构体的实体会在file_operations的mmap()成员函数里被赋值给相应的vma->vm_ops，而上述open()函数也通常在mmap()里调用，close()函数会在用户调用munmap()的时候被调用到。如下代码给出了vm_operations_struct的操作范例：
```c
static int xxx_mmap(struct file *filp, struct vm_area_struct *vma)
{
	if (remap_pfn_range(vma, vma->start, vma->pgoff, vma->end - 
		vma->start, vma->vm_page_prot)); /* 建立页表 */
		return -EAGAIN;
	vma->vm_ops = &xxx_remap_vm_ops;
	xxx_vma_open(vma);
	return 0;
}

static void xxx_vma_open(struct vm_area_struct *vma) /* VMA打开函数 */
{
	...
	printk(KERN_NOTICE "xxx VMA open, virt %lx, phys %lx\n", vma->vm-start,
		vma->vm_pgoff << PAGE_SHIFT);
}

static void xxx_vma_close(struct vm_area_struct *vma) /* VMA关闭函数 */
{
	...
	printk(KERN_NOTICE "xxx VMA close\n");
}

/* VMA操作结构体 */
static struct vm_operations_struct xxx_remap_vm_ops = {
	.open = xxx_vma_open,
	.close = xxx_vma_close,
	...
};
```
调用remap_pfn_range()创建页表项，以VMA结构体的成员（VMA的数据成员是内核根据用户的请求自己填充的）作为remap_pfn_range()参数，映射的虚拟地址范围是vma->vm_start至vma->vm_end。

remap_pfn_range()函数原型如下：
```c
int remap_pfn_range(struct vm_area_struct *vma, unsigned long addr,
		unsigned long pfn, unsigned long size, pgprot_t prot);
```
其中的addr参数表示内存映射开始处的虚拟地址，remap_pfn_range()为addr~addr+size的虚拟地址构造页表。

pfn是虚拟地址应该映射到的物理地址的页帧号，实际上就是物理地址右移PAGE_SHIFT位。若PAGE_SIZE为4K，则PAGE_SHIFT就为12，因为PAGE_SIZE等于1<<PAGE_SHIFT。

prot是新页所要求的保护属性。

在驱动程序中，我们能使用remap_pfn_range()映射内存的保留页、设备I/O、framebuffer、camera等内存。在remap_pfn_range()上又可以进一步封装除io_remap_pfn_range()、vm_iomap_memory()等API。

```c
#define io_remap_pfn_range remap_pfn_range
int vm_iomap_memory(struct vm_area_struct *vma, phys_addr_t start, unsigned long len)
{
	unsigned long vm_len, pfn, pages;
	...
	len += start & ~PAGE_MASK;
	pfn = start >> PAGE_SHIFT;
	pages = (len + ~PAGE_MASK) >> PAGE_SHIFT;
	...
	pfn += vma->vm_pgoff;
	pages -= vma->vm_pgoff;
	/* Can we fit all of the mapping */
	vm_len = vma->vm_end - vma->vm_start;
	...
	/* Ok, let it rip */
	return io_remap_pfn_range(vma, vma->start, pfn, vm_len, vma->vm_page_prot);
}
```

下面代码给出了LCD驱动映射framebuffer的物理地址到用户空间的典型范例，代码取自drivers/video/fbdev/core/fbmem.c：
```c
static int
fb_mmap(struct file *file, struct vm_area_struct *vma)
{
	struct fb_info *info = file_fb_info(file);
	struct fb_ops *fb;
	unsigned long mmio_pgoff;
	unsigned long start;
	u32 len;

	if (!info)
		return -ENODEV;
	fb = info->fbops;
	if (!fb)
		return -ENODEV;
	mutex_lock(&info->mm_lock);
	if (fb->fb_mmap) {
		int res;
		res = fb->fb_mmap(info, vma);
		mutex_unlock(&info->mm_lock);
		return res;
	}

	/* 
	 * Ugh. This can be either the frame buffer mapping, or
	 * if pgoff points pass it, the mmio mapping.
	 */
	start = info->fix.smem_start;
	len = info->fix.smem_len;
	mmio_pgoff = PAGE_ALIGH((start & ~PAGE_MAKS) + len) >> PAGE_SHIFT;
	if (vma->vm_pgoff >= mmio_pgoff) {
		if (info->var.accel_flags) {
			mutex_unlock(&info->mm_lock);
			return -EINVAL;
		}

		vma->vm_pgoff -= mmio_pgoff;
		start = info->fix.mmio_start;
		len = info->fix.mmio_len;
	}
	mutex_unlock(&info->mm_lock);

	vma->vm_page_prot = vm_get_page_prot(vma->vm_flags);
	fb_pgprotect(file, vma, start);

	return vm_iomap_memory(vma, start, len);
}
```
通常，I/O内存被映射时需要时nocache的，这时候，我们应该对vma->vm_page_prot设置nocache标志后再映射，如下面代码所示：
```c
static int xxx_nocache_mmap(struct file *filp, struct vm_area_struct *vma)
{
	vma->vm_page_prot = pgprot_noncached(vma->vm_page_prot); /* 赋nocache标志 */
	vma->vm_pgoff = ((u32)map_start >> PAGE_SHIFT);
	/* 映射 */
	if (remap_pfn_range(vma, vma->vm_start, vma->vm_pgoff, vma->vm_end -
		vma->vm_start, vma->vm_page_prot));
		return -EAGAIN;
	return 0;
}
```
上述代码pgprot_noncached()是一个宏，它高度依赖于CPU体系结构，ARM的pgprot_noncached()定义如下：
```c
#define pgprot_noncached(prot) \
		__pgprot_modify(prot, L_PTE_MT_MASK, L_PTE_MT_UNCACHE)
```
另一个比pgprot_noncached()宏少一些限制的宏是pgprot_writecombine()，它的定义如下：
```c
#define pgprot_writecombine(prot) \
		__pgprot_modify(prot, L_PTE_MT_MASK, L_PTE_MT_BUFFERABLE)
```

pgprot_noncached()实际禁止了相关页的Cache和写缓冲（Write Buffer），pgprot_writecombine()则没有禁止写缓冲。ARM的写缓冲器是一个非常小的FIFO存储器，位于处理器核与主存之间，其目的在于将处理器核和Cache从较慢的主存写操作中解脱出来。写缓冲区与Cache在存储层次上处于同一层次，但是它只作用于写主存。

### 2. fault()函数
除了remap_pfn_range()以外，在驱动程序中实现VMA的fault()函数通常可以为设备提供更加灵活内存映射途径。当访问的额页不在内存里，即发生缺页异常时，fault()会被内核自动调用，而fault()具体行为可以自定义。这是因为当发生缺页异常时，系统会经过如下处理过程。

> 1. 找到缺页的虚拟地址所在的VMA  
> 2. 如果必要，分配中间页目录表和页表  
> 3. 如果页表项对应的物理页面不存在，则调用这个VMA的fault()方法，它返回物理页面的页描述符  
> 4. 将物理页面的地址填充到页表中  

fault()函数在早期内核版本中被定义为nopage()，后来变更为了fault()。

下面代码给出驱动程序中使用fault()的范例：
```c
static int xxx_fault(struct vm_area_struct *vma, struct vm_fault *vmf)
{
	unsigned long paddr;
	unsigned long pfn;
	pgoff_t index = vmf->pgoff;
	struct vma_data *vdata = vam->vm_private_data;
	
	...

	pfn = paddr >> PAGE_SHIFT;

	vm_insert_pfn(vma, (unsigned long)vmf->virtual_address, pfn);

	return VM_FAULT_NOPAGE;
}
```

大多数设备驱动都不需要提供设备内存到用户空间的映射能力，因为，对于串口等面向流的设备而言，实现这种映射毫无意义。而对于显示、视频等设备，建立映射可减少用户空间和内核空间之间的内存复制。