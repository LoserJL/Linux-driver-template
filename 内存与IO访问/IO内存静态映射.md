# I/O内存静态映射
在将Linux移植到目标板的过程中，有得会建立外设I/O内存物理地址到虚拟地址的静态映射，这个映射通过在于电路板对应的map_desc结构体数组中添加新的成员来完成，map_desc的定义如下：
```c
struct map_desc {
	unsigned long virtual;	/* 虚拟地址 */
	unsigned long pfn;		/* __phys_to_pfn(phy_addr) */
	unsigned long length;	/* 大小 */
	unsigned int type;		/* 类型 */
};
```
例如，在内核arch/arm/mach-ixp2000/ixdp2x01.c文件对应的Intel IXDP2401和IXDP2801平台上包含一个CPLD，该文件中就进行了CPLD物理地址到虚拟地址的静态映射，代码如下：
```c
static struct map_desc ixdp2x01_io_desc __initdata = {
	.virtual	= IXDP2X01_VIRT_CPLD_BASE,
	.pfn		= __phys_to_pfn(IXDP2X01_PHYS_CPLD_BASE),
	.length		= IXDP2X01_CPLD_REGION_SIZE,
	.type		= MT_DEVICE,
};

static void __init ixdp2x01_map_io(void)
{
	ixp2000_map_io();
	iotable_init(&ixdp2x01_io_desc, 1);
}
```
iotable_init()函数是最终建立页映射的函数，它通过MACHINE_START、MACHINE_END宏赋值给电路板的map_io()函数。将Linux操作系统移植到特定平台上，MACHINE_START(或者DT_MACHINE_START)、MACHINE_END宏之间的定义针对特定电路板而设计，其中的map_io()成员函数完成I/O内存的静态映射。

在一个已经移植好操作系统的内核中，驱动工程师可以对非常规内存区域的I/O内存（外设控制器寄存器、MCU内部集成的外设控制器寄存器等）依照电路板的资源使用情况添加到map_desc数组中，但是目前该方法已经不值得推荐。