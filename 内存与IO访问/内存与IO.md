# 内存与IO
由于Linux系统提供了复杂的内存管理功能，所以内存的概念在Linux中相对复杂，有常规内存、高端内存、虚拟地址、逻辑地址、总线地址、物理地址、I/O内存、设备内存、预留内存等概念。

# CPU与内存、I/O
## 1. 内存空间与I/O空间
在x86处理器中存在着I/O空间的概念，I/O空间是相对内存空间而言的，它通过特定的指令in、out来访问。端口号标识了外设的寄存器地址。Intel语法中的in、out指令格式如下：
```
IN 累加器, {端口号|DX}
OUT {端口号|DX}，累加器
```

目前，大多数嵌入式微控制器（如ARM、PowerPC等）中并不提供I/O空间，而仅存在内存空间。内存空间可以直接通过地址、指针来访问，程序及在程序运行中使用的变量和其它数据都存在与内存空间中。

内存地址可以直接由C语言指针操作，如在186处理器中执行如下代码：
```c
unsigned char *p = (unsigned char *)0xF000FF00;
*p = 11;
```
以上程序的意义是在绝对地址0xF0000+0xFF00（186处理器使用16位段地址和16为偏移地址）中写入11。

而在ARM、PowerPC等未采用段地址的处理器中，p指向的内存空间就是0xF000FF00，而*p = 11就是在该地址写入11。

再如，186处理器启动后会在绝对地址0xFFFF0（对应的C语言指针是0xF000FFF0，0xF000为段地址，0xFFF0为段内偏移）中执行，请看下面的代码：
```c
typedef void (*lpFunction)();	/* 定义一个无参数，无返回类型的函数指针类型 */
lpFuntion lpReset = (lpFunction)0xF000FFF0; /* 定义一个函数指针，指向CPU启动后所执行的第一条指令的位置 */
lpReset();		/* 调用函数 */
```
在以上程序中，没有定义任何一个函数实体，但是程序却执行了函数调用：lpReset()，它实际上起了软重启的作用，跳转到CPU启动后第一条要执行的指令的位置。因此，可以通过函数指针调用一个没有函数体的“函数”，这本质上只是换一个地址开始执行。

即便是在x86处理器中，虽然提供了I/O空间，如果由我们自己设计电路板，外设仍然可以只挂接在内存空间中。此时，CPU可以像访问内存单元那样访问外设I/O端口，而不需要设立专门的I/O指令。因此，内存空间是必需的，而I/O空间是可选的。

# 2. 内存管理单元
高性能处理器一般会提供一个内存管理单元（MMU），该单元辅助操作系统进行内存管理，提供虚拟地址和物理地址的映射、内存访问权限保护和Cache缓存控制等硬件支持。操作系统内核借助MMU可以让用户感觉到可以使用非常大的内存空间，从而使得编程人员在写程序时不用考虑实际物理内存的实际容量。

为了理解基本的MMU操作原理，需先明晰几个概念。

>1. TLB(Translation Lookaside Buffer)：即转换旁路缓存，TLB是MMU的核心部件，它缓存少量的虚拟地址与物理地址的转换关系，是转换表的Cache，因此也经常被称为“快表”。  
>2. TTW(Translation Table walk)：即转换表漫游，当TLB中没有缓存对应的地址转换关系时，需要通过对内存中转换表（大多数处理器的转换表为多级页表）的访问来获得虚拟地址与物理地址的对应关系。TTW成功后，结果应写入TLB中。

在ARM处理中，当要访问存储器时，MMU先查找TLB中的虚拟地址表。如果ARM的结构支持分开的数据TLB(DTLB)和指令TLB(ITLB)，则除了取指令使用ITLB外，其他的都是用DTLB。  
若TLB中没有虚拟地址的入口，则转换表遍历硬件并从存放于主存储器内的转换表中获取地址转换信息和访问权限（即执行TTW），同时将这些信息放入TLB，它或者放在一个没有使用的入口或者替换一个已经存在的入口。之后，在TLB条目中控制信息的控制下，当访问权限允许时，对真实物理地址的访问将在Cache或者内存中发生。

ARM内TLB条目中的控制信息用于控制对对应地址的访问权限以及Cache的操作。  
>C(高速缓存)和B(缓冲)位被用来控制对应地址的高速缓存和写缓冲，并决定是否进行高速缓存。  
>访问权限和域位用来控制读写访问是否被允许。如果不允许，MMU将向ARM处理器发送一个存储器异常，否则访问将被允许进行。

上述描述的MMU机制虽然针对ARM，但PowerPC、MIPS等处理器均有类似操作。

MMU具有虚拟地址和物理地址转换、内存访问权限保护等功能，这将使得Linux操作系统能单独为系统的每个用户进程分配独立的内存空间并保证用户空间不能访问内核空间的地址，为操作系统的虚拟内存管理模块提供硬件基础。

在Linux 2.6.11之前，Linux内核硬件无关层使用了三级页表PGD、PMD和PTE；从Linux 2.6.11开始，为了配合64位CPU的体系结构，硬件无关层则使用了4级页表目录管理的方式，即PGD、PUD、PMD和PTE。注意，这只是一种软件意思上的抽象，实际硬件的页表级数可能小于4。

如下代码给出了一个典型的从虚拟地址得到PTE的页表查询（Page Table Walk）过程，它取自arch/arm/lib/uaccess_with_memcpy.c：
```c
static int
pin_page_for_write(const void __user *_addr, pte_t **ptep, spinlock_t **ptlp)
{
	unsigned long addr = (unsgined long)_addr;
	pgd_t *pgd;
	pmd_t *pmd;
	pte_t *pte;
	pud_t *pud;
	spinlock_t *ptl;

	pgd = pgd_offset(current->mm, addr);
	if (unlikely(pgd_none(*pgd) || pgd_bad(*pgd)))
		return 0;

	pud = pud_offset(pgd, addr);
	if (unlikely(pud_none(*pgd) || pud_bad(*pgd)))
		return 0;

	pmd = pmd_offset(pud, addr);
	if (unlikely(pmd_none(*pgd) || pmd_bad(*pgd)))
		return 0;

	/*
	 * A pmd can be bad if it refers to a HugeTLB or THP page.
	 *
	 * Both THP and HugeTLB pages have the same pmd layout
	 * and should not be manipulated by the pte functions.
	 *
	 * Lock the page table for the destination and check
	 * to see that it's still huge and whether or not we will
	 * need to fault on write, or if we have a splitting THP.
	 */
	if (unlikely(pmd_thp_or_huge(*pmd))) {
		ptl = &current->mm->page_table_lock;
		spin_lock(ptl);
		if (unlikely(!pmd_thp_or_huge(*pmd)
			|| pmd_hugewillfault(*pmd)
			|| pmd_trans_splitting(*pmd))) {
			spin_unlock(ptl);
			return 0;
		}

		*ptep = NULL;
		*ptlp = ptl;
		return 1;
	}

	if (unlikely(pmd_bad(*pmd)))
		return 0;

	pte = pte_offset_map_lock(current->mm, pmd, addr, &ptl);
	if (unlikely(!pte_present(*pte) || !pte_young(*pte) || 
		!pte_write(*pte) || !pte_dirty(*pte))) {
		pte_unmap_unlock(pte, ptl);
		return 0;
	}

	*ptep = pte;
	*ptlp = ptl;

	return 1;
}
```
current类型为task_struct，其中类型为mm_struct的mm用于描述Linux进程所占有的内存资源。上述代码中的pgd_offset、pud_offset、pmc_offset分别用于得到一级页表、二级页表、三级页表的入口，最后通过pte_offset_map_lock()得到目标页表项pte。而且代码中还通过pmd_thp_or_huge()判断是否有巨页的情况，如果是巨页，就直接访问pmd。

但是，MMU并不是对所以的处理器都是必须的，新版的Linux 2.6支持不带MMU的处理器。在嵌入式系统中，仍存在大量没有MMU的处理器，Linux 2.6为了更广泛地应用于嵌入式系统，融合了mClinux，以支持那些没有MMU的处理器。