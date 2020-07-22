# Linux内存管理
对于包含MMU的处理器而言，Linux提供了复杂的存储管理系统，使得进程所能访问的内存达到4GB。

在Linux系统中，进程的4GB内存空间被分为两个部分——用户空间与内核空间，用户空间的地址一般分布于0~3GB(即PAGE_OFFSET,在x86中它等于0xC0000000)，这样，剩下的3~4GB为内核空间。用户进程通常只能访问用户空间的虚拟地址，不能访问内核空间的虚拟地址。用户进程通常只有通过系统调用（代表用户进程在内核态执行）等方式才能访问到内核空间。

每个进程的用户空间都是相互独立、互不相干的，用户进程各自有不同的页表。而内核空间是由内核负责映射，它并不会跟着进程改变，是固定的。内核空间的虚拟地址到物理地址的映射是被所有进程共享的，内核的虚拟空间独立于其他程序。

Linux中的1GB内核地址空间又被划分为物理内存映射区、虚拟内存分配区、高端页面映射区、专用页面映射区和系统保留映射区这几个区域。

对于x86系统而言，一般情况下，物理内存映射区最大长度为896MB，系统的物理内存被顺序映射在内核空间的这个区域中。当系统物理内存大于896MB时，超过物理内存映射区的那部分内存成为高端内存（未超过的部分通常称为常规内存），内核在存取高端内存时，必须将它们映射到高端页面映射区。

Linux保留内核空间最顶部FIXADDR_TOP~4GB的区域作为保留区。

紧接着最顶端的保留区以下的一段区域为专用页面映射区（FIXADDR_START~FIXADDR_TOP），它的总尺寸和每一页的用途由fixed_address枚举结构在编译时预定义，用__fix_to_virt(index)可获取专用区内预定于页面的逻辑地址。其开始地址和结束地址宏定义如下：
```c
#define FIXADDR_START		(FIXADDR_TOP - __FIXADDR_SIZE)
#define FIXADDR_TOP			((unsigned long)__FIXADDR_TOP)
#define __FIXADDR_TOP		0xfffff000
```

接下来，如果系统配置了高端内存，则位于专用页面映射区之下的就是一段高端内存映射区，其起始地址为PKMAP_BASE，定义如下：
```c
#define PKMAP_BASE	( (FIXADDR_BOOT_START - PAGE*(LAST_PKMAP + 1)) & PMD_MASK)
```
其中所涉及的宏定义如下：
```c
#define FIXADDR_BOOT_START	(FIXADRR_TOP - __FIXADDR_BOOT_SIZE)
#define LAST_PKMAP	PTRS_PER_PTE
#define PTRS_PER_PTE	512
#define PMD_MASK	(~(PMD_SIZE-1))
#define PMD_SIZE	(1UL << PMD_SHIFT)
#define PMD_SHIFT	21
```

在物理区和高端内存映射区之间为虚拟内存分配器区（VMALLOC_START~VMALOOC_END），用于vamlloc()函数，它的前部与物理内存映射区有一个隔离带，后部与高端映射区也有一个隔离带，vmalloc区域定义如下：
```c
#define VMALLOC_OFFSET	(8*1024*1024)
#define VMALLOC_START	(((unsigned long)high_memory +
	vmalloc_earlyreserve + 2*VMALLOC_OFFSET-1) & ~(VMALLOC_OFFSET-1))
#ifdef	CONFIG_HIGHMEM		/* 支持高端内存 */
#define VMALLOC_END		(PKMAP_BASE-2*PAGE_SIZE)
#else						/* 不支持高端内存 */
#define VMALLOC_END		(FIXADDR_START-2*PAGE_SIZE)
#endif
```

当系统物理内存高于4GB时，必须使用CPU的扩展分页（PAE）模式所提供的64位页目录项才能存取到4GB以上的物理内存，这需要CPU的支持。加入了PAE功能的Intel Pentium Pro及以后的CPU允许内存最大可配置到64GB，它们具备36位物理地址空间寻址能力。

由此可见，对于32位的x86而言，在3~4GB之间的内核空间中，从低地址到高地址依次为：物理地址映射区->隔离带->vmalloc虚拟内存分配器区->隔离带->高端内存映射区->专用页面映射区->保留区。

直接进行映射的896MB的物理内存其实又分为两个区域，在低于16MB的区域，ISA设备可以做DMA，所以该区域为DMA区域（内核为了保证ISA驱动在申请DMA缓冲区的时候，通过GFP_DMA标记可以确保申请到16MB以内的内存，所以必须把这个区域列为一个单独的区域管理）；16MB~896MB之间为常规区域，高于896MB的就是高端内存区域了。

32位ARM Linux的内核空间地址映射与x86不太一样，内核文档Documentation/arm/memory.txt给出了ARM Linux的内存映射情况。0xffff0000~0xffff0fff是“CPU vector page”，即向量表的地址。0xffc00000~0xffefffff是DMA内存映射区域，dma_alloc_xxx族函数把DMA缓冲区映射在这一段，VMALLOC_START~VMALLOC_END-1是vmalloc和ioremap区域（在vmalloc区域的大小可以配置，通过"vmalloc="这个启动参数可以指定），PAGE_OFFSET~high_memory-1是DMA和正常区域的映射区域，MODULES_VADDR~MODULES_END-1是内核模块区域，PKMAP_BASE~PAGE_OFFSET-1是高端内存映射区。假设我们把PAGE_OFFSET定义为3GB，实际上Linux内核模块位于3GB-16MB~3GB-2MB，高端内存映射区则通常位于3GB-2MB~3GB，这里假定编译内核的时候选择的是VMSPLIT_3G（3G/1Guser/kernel split），如果选择的是VMSPLIT_2G（2G/2G user/kernel split），则Linux内核模块位于2GB-16MB~2GB-2MB，高端内存映射区则通常位于2GB-2MB~2GB。

ARM系统的Linux之所以把内核模块安置在3GB或者2GB附近的16M范围内，主要是为了实现内核模块和内核本身的代码段之间的短跳转。

对于ARM SoC而言，如果芯片内部有的硬件组件的DMA引擎访问内存时有地址空间限制（某些空间访问不到），比如假设UART控制器的DMA只能访问32MB，那么这个低32MB就是DMA区域；32MB到高端内存地址的这段称为常规区域；再之上的称为高端内存区域。

DMA、常规、高端内存区域可能的分布有如下几种：
>1. 有硬件的DMA引擎不能访问全部地址，且内存较大而无法全部在内核空间虚拟地址映射下，存放有3个区域，即DMA区域->常规区域->高端内存区域  
>2. 没有硬件的DMA引擎不能访问全部地址，且内存较大而无法全部在内核空间虚拟地址映射下，则常规区域实际退化为0，存放有2个区域，即DMA区域->高端内存区域  
>3. 有硬件的DMA引擎不能访问全部地址，且内存较小可以全部在内核空间虚拟地址映射下，则高端内存区域实际退化为0，存放有2个区域，即DMA区域->常规区域  
>4. 没有硬件的DMA引擎不能访问全部地址，且内存较小可以全部在内核空间虚拟地址映射下，则常规区域与高端内存区域实际退化为0，即只有DMA区域

DMA、常规、高端内存这3个区域都采用buddy算法进行管理，把空闲的页面以2的n次方为单位进行管理，因此Linux最底层的内存申请都是以2的n次方为单位的。Buddy算法最主要的优点是避免了外部碎片，任何时候区域内的空闲内存都能以2的n次方进行拆分或合并。

对于内核物理内存映射区的虚拟内存（即从DMA和常规区域映射过来的），使用virt_to_phys()可以实现内核虚拟地址转化为物理地址，与之对应的函数phys_to_virt()将物理地址转化为内核虚拟地址。

**注意：** 上述virt_to_phys()和phys_to_virt()方法仅适用于DMA和常规区域，高端内存的虚拟地址与物理地址之间不存在如此简单的换算关系。

# 内存存取
## 1. 用户空间内存动态申请
在用户空间中动态申请内存的函数为malloc()，这个函数在各种操作系统上使用是一致的，malloc()申请的内存释放函数是free()。对于Linux而言，C库的malloc()函数一般通过brk()和mmap()两个系统调用从内核申请内存。

由于用户空间C库的malloc()算法实际上具备一个二次管理能力，所以并不是每次申请和释放内存都一定伴随着对内核的系统调用。比如，如下代码可以从内核拿到内存后，立即调用free()，由于free()之前调用了mallopt(M_TRIM_THRESHOLD, -1)和mallopt(M_MMAP_MAX, 0)，这个free()并不会把内存还给内核，而只是还给了C库的分配算法（内存仍然属于这个进程），因此之后的所有的内存动态申请和释放都在用户态下进行。
```c
#include <malloc.h>
#include <sys/mman.h>

#define SOMESIZE	(100*1024*1024)		//100MB

int main(int argc, char *argv[])
{
	unsigned char *buffer;
	int i;

	if (mlockall(MCL_CURRENT | MCL_FUTURE))
		mallopt(M_TRIM_THRESHOLD, -1);
	mallopt(M_MMAP_MAX, 0);

	buffer = malloc(SOMESIZE);
	if (!buffer)
		exit(-1);

	/*
	 * Touch each page in this piece of memory to get it
	 * mapped into RAM
	 */
	for (i = 0; i < SOMESIZE; i += page_size)
		buffer[i] = 0;
	free(buffer);
	/* <do your RT-thing> */

	return 0;
}
```

另外，Linux内核总是采用按需调页（Demand Paging），因此当malloc()返回的时候，虽然是成功返回，但是内核并没有真正给这个进程内存，这个时候如果去读申请的内存，内容全部是0，这个页面的映射是只读的。只有当写到这个页面的时候，内核才在页错误后，真正把这个页面给这个进程。

## 2. 内核空间内存动态申请
在Linux内核空间中申请内存涉及的函数主要包括kmalloc()、__get_free_pages()和vmalloc()等。kmalloc()和__get_free_pages()（及其类似函数）申请的内存位于DMA和常规区域的映射区，而且在物理上也是连续的，它们与真实的物理地址只有一个固定的偏移，因此存在较简单的转换关系。而vmalloc()在虚拟内存空间给出一块连续的内存区，实质上，这块连续的虚拟内存在物理内存中并不一定连续，而vmalloc()申请的虚拟内存和物理内存之间也没有简单的换算关系。

### 1. kmalloc()
```c
void *kmalloc(size_t size, int flags);
```
kmalloc()的第一个参数是要分配的块的大小，第二个参数为分配标志，用于控制kmalloc()的行为。

最常用的分配标志是GFP_KERNEL，其含义是在内核空间的进程中申请内存。kmalloc()的底层依赖于__get_free_pags()来实现，分配标志的前缀GFP正好是这个底层函数的缩写。使用GFP_KERNEL申请内存时，若暂时不能满足，则进程会睡眠等待页，即会引起阻塞，因此不能在中断上下文或持有自旋锁的时候使用GFP_KERNEL申请内存。

由于在中断处理函数、tasklet和内核定时器等非进程上下文中不能阻塞，所以此时驱动应当使用GFP_ATOMIC申请内存。当使用GFP_ATOMIC标志申请内存时，若不能满足，则不等待，直接返回。

其他的申请标志还包括：
```
GFP_USER		//用来为用户空间页分配内存，可能阻塞
GFP_HIGHUSER	//类似GFP_USER，但是它从高端内存分配
GFP_DMA			//从DMA区域分配内存
GFP_NOIO		//不允许任何I/O初始化
GFP_NOFS		//不允许进行任何文件系统调用
__GFP_HIGHMEM	//指示分配的内存可以位于高端内存
__GFP_COLD		//请求一个较长时间不访问的页
__GFP_NOWARN	//当一个分配无法满足时，阻止内核发出警告
__GFP_HIGH		//高优先级请求，允许获得被内核保留给紧急状况使用的最后的内存页
__GFP_REPEAT	//分配失败，则尽力重复尝试
__GFP_NOFAIL	//只许申请成功，不推荐
__GFP_NORETRY	//若申请不到，则立即放弃
```
使用kmalloc()申请的内存应使用kfree()释放，这个函数的用法和用户空间的free()一致。

### 2. __get_free_pages()
__get_free_pages()系列函数/宏本质上是Linux内核最底层用于获取空闲内存的方法，因为最底层的buddy算法以2的n次方管理空闲内存，所以最底层的内存申请总是以2的n次方为单位的。

__get_free_pages()系列函数/宏包括get_zeroed_page()、__get_free_page()和__get_free_pages()。

```c
get_zeroed_page(unsigned int flags);
```
该函数返回一个指向新页的指针并将该页清零。

```c
__get_free_page(unsigned int flags);
```
该宏返回一个指向新页的指针，但是不清零，它实际上是：
```c
#define __get_free_page(gfp_mask) \
		__get_free_pages((gfp_mask), 0)
```
就是调用了下面的__get_free_pages()申请了一页。
```c
__get_free_pages(unsigned int flags, unsigned int order);
```
该函数可以分配多个页，并返回分配内存的首地址，分配的页数为2的order次方，分配的页也不清零。order允许的最大值是10（即1024页），或者11（即2048页），这取决于具体的硬件平台。

__get_free_pages()和get_zeroed_page()在实现中调用了alloc_pages()函数，alloc_pages()既可以在内核空间分配，也可以在用户空间分配，其原型为：
```c
struct page *alloc_pages(int gfp_mask, unsigned long order);
```
参数含义与__get_free_pages()类似，但它返回第一个页的描述符而非首地址。

使用__get_free_pages()系列函数/宏申请的内存应使用下列函数释放：
```c
void free_page(unsigned long addr);
void free_pages(unsigned long addr, unsigned long order);
```
__get_free_pages()等函数在使用时，其申请标志的值与kmalloc()完全一样，各标志的含义也与kmalloc()一样，最常用的时GFP_KERNEL和GFP_ATOMIC。

### 3. vmalloc()
vmalloc()一般只为存在于软件中（没有对应的硬件意义）的较大的顺序缓冲区分配内存，vmalloc()远大于__get_free_pages()的开销，为了完成vmalloc()，新的页表项需要被建立，因此，使用vmalloc()分配少量内存（1页以内）是不妥的。
vmalloc()申请的内存使用vfree()释放，二者原型如下：
```c
void *vmalloc(unsigned long size);
void vfree(void *addr);
```
vmalloc()不能用在原子上下文中，因为它的内部实现使用了标志位GFP_KERNEL的kmalloc()。

使用vmalloc()函数的一个例子函数是create_moudle()系统调用，它利用vmalloc()函数来获取被创建模块需要的内存空间。

vmalloc()在申请内存时，会进行内存的映射，改变页表项，不像kmalloc()实际用的是开机过程中就映射好的DMA和常规区域的页表项，因此vmalloc()的虚拟地址和物理地址不是一个简单的线性映射。

### 4. slab与内存池
一方面，完全使用以页为单位申请和释放内存容易导致浪费（如果需要少量字节，也需要申请1页）；另一方面，在操作系统的运作过程中，经常会涉及大量对象的重复生成、使用和释放内存问题。在Linux系统中所用到的对象，比较典型的例子是inode、task_struct等。如果我们能够用合适的方法使得对象在前后两次使用时分配在同一块内存或同一类内存空间且保留了基本的数据结构，就可以大大提高效率。slab算法就是针对上述特点设计的，实际上kmalloc()就是用slab机制实现的。

slab是建立在buddy算法之上的，它从buddy算法拿到2的n次方页面后进行二次管理，这一点和用户空间的C库很像。slab申请的内存以及基于slab的kmalloc()申请的内存，与物理内存之间也是一个简单的线性偏移。

##### (1) 创建slab缓存
```c
struct kmem_cache *kmem_cache_create(const char *name, size_t size
	size_t align, unsigned long flags,
	void (*ctor)(void *, struct kmem_cache, unsigned long),
	void (*dtor)(void *, struct kmem_cache, unsigned long));
```
kmem_cache_create()用于创建slab缓存，它是一个可以保留任意数目且全部同样大小的后备缓存。参数size是要分配的每个数据结构的大小，参数flags是控制如何进行分配的位掩码，包括SLAB_HWCACHE_ALIGH(每个数据对象被对齐到一个缓冲行)、SLAB_CACHE_DMA(要求数据对象在DMA区域中分配)等。

##### (2) 分配slab缓存
```c
void *kmem_cache_alloc(struct kmem_cache *cachep, gfp_t flags);
```
上述函数在kmem_cache_create()创建的slab后备缓存中分配一块并返回首地址指针。

##### (3) 释放slab缓存
```c
void kmem_cache_free(struct kmem_cache *cachep, void *objp);
```
上述函数释放由kmem_cache_alloc()分配的缓存。

##### (4) 收回slab缓存
```c
int kmem_cache_destroy(struct kmem_cache *cachep);
```

下面代码是slab缓存使用范例：
```c
/* 创建slab缓存 */
static kmem_cache_t *xxx_cachep;
xxx_cachep = kmem_cache_create("xxx", sizeof(struct xxx),
				0, SLAB_HWCACHE_ALIGH | SLAB_PANIC, NULL, NULL);
/* 分配slab缓存 */
struct xxx *ctx;
ctx = kmem_cache_alloc(xxx_cachep, GFP_KERNEL);
... /* 使用slab缓存 */
/* 释放slab缓存 */
kmem_cache_free(xxx_cachep, ctx);
kmem_cache_destroy(xxx_cachep);
```

在系统中，通过proc/slabinfo节点可以获知当前slab的分配和使用情况。

注意：slab不是要替代__get_free_pages()，其在最底层仍然依赖于__get_free_pages()，slab在最底层每次申请1页或多页，之后再分隔这些页为更小的单元进行管理，从而节省了内存，也提高了slab缓冲对象的访问效率。

除了slab以外，内核中还支持内存池，内存池技术也是一种非常经典的用于分配大量小对象的后备缓存技术。
在Linux内核中，与内存池相关的操作有：
##### (1) 创建内存池
```c
mempool_t *mempool_create(int min_nr, mempool_alloc_t *alloc_fn,
	mempool_free_t *free_fn, void *pool_data);
```
mempool_create()函数用于创建一个内存池，min_nr参数是需要预分配对象的数目，alloc_fn和free_fn是指向内存池机制提供的标准对象分配和回收函数的指针，其原型分别为：
```c
typedef void *(mempool_alloc_t)(int gfp_mask, void *pool_data);
```
和
```c
typedef void (mempool_free_t)(void *element, void *pool_data);
```
pool_data是分配和回收函数用到的指针，gfp_mask是分配标记。只有当__GFP_WAIT标记被指定时，分配函数才会休眠。

##### (2)分配和回收对象
在内存池中分配和回收对象需要如下函数完成：
```c
void *mempool_alloc(mempool_t *pool, int gfp_mask);
void mempool_free(void *element, mempool_t *pool);
```
mempool_alloc()用来分配对象，如果内存池分配器无法提供内存，那么就可以用预分配的池。

##### (3) 回收内存池
```c
void mempool_destroy(mempool_t *pool);
```
回收由mempool_create()创建的内存池。