# 一. platform驱动介绍
Linux2.6开始引入了platform模型，在platform设备驱动模型中，需关心总线、设备和驱动这3个实体，总线将设备和驱动绑定。在系统每注册一个设备的时候，会寻找与之匹配的驱动；相反的，在系统每注册一个驱动的时候，会寻找与之匹配的设备，而匹配由总线完成。  
一个现实的Linux设备和驱动通常都需要挂接在一种总线上，对于本身依附于PCI、USB、I2C、SPI等的设备而言，这自然不是问题，但是在嵌入式系统里面，在SoC系统中集成的独立外设控制器、挂接在SoC内存空间的外设等却不依附于此类总线。基于这一背景，Linux发明了一种虚拟的总线，称为platform总线，相应的设备称为platform_device，而驱动成为platform_driver。
>*注意：所谓的platform_device并不是与字符设备、块设备和网络设备并列的概念，而是Linux系统提供的一种附加手段，例如，我们通常把在SoC内部集成的I2C、RTC、LCD、看门狗等控制器都归纳为platform_device，而它们本身就是字符设备。*

platform_device结构体的定义在include/linux/platform_device.h中，如下所示：
```c
struct platform_device {
	const char	*name;
	int		id;
	bool		id_auto;
	struct device	dev;
	u32		num_resources;
	struct resource	*resource;

	const struct platform_device_id	*id_entry;
	char *driver_override; /* Driver name to force a match */

	/* MFD cell pointer */
	struct mfd_cell *mfd_cell;

	/* arch specific additions */
	struct pdev_archdata	archdata;
};
```

platform_driver结构体定义在include/linux/platform_device.h中，如下所示：
```c
struct platform_driver {
	int (*probe)(struct platform_device *);
	int (*remove)(struct platform_device *);
	void (*shutdown)(struct platform_device *);
	int (*suspend)(struct platform_device *, pm_message_t state);
	int (*resume)(struct platform_device *);
	struct device_driver driver;
	const struct platform_device_id *id_table;
	bool prevent_deferred_probe;
};
```
platform_driver这个结构体中包含probe()、remove()、一个device_driver实例、电源管理函数suspend()、resume()  
直接填充platform_driver的suspend()、resume()做电源管理回调的方法目前已经过时，较好的做法是实现platform_driver的device_driver中的dev_pm_ops结构体成员（参见Linux电源管理）

对于platform_driver中的device_driver，其定义在include/linux/device.h中，具体定义如下：
```c
struct device_driver {
	const char		*name;
	struct bus_type		*bus;

	struct module		*owner;
	const char		*mod_name;	/* used for built-in modules */

	bool suppress_bind_attrs;	/* disables bind/unbind via sysfs */

	const struct of_device_id	*of_match_table;
	const struct acpi_device_id	*acpi_match_table;

	int (*probe) (struct device *dev);
	int (*remove) (struct device *dev);
	void (*shutdown) (struct device *dev);
	int (*suspend) (struct device *dev, pm_message_t state);
	int (*resume) (struct device *dev);
	const struct attribute_group **groups;

	const struct dev_pm_ops *pm;

	struct driver_private *p;
};
```

>**与platform_driver地位对等的i2c_driver、spi_driver、usb_driver、pci_driver中都包含了device_driver结构体实例成员。它其实描述了各种xxx_driver（xxx是总线名）在驱动意义上的一些共性。**

系统为platform总线定义了一个bus_type的实例platform_bus_type，其定义位于drivers/base/platform.c中，具体如下：
```c
struct bus_type platform_bus_type = {
	.name		= "platform",
	.dev_groups	= platform_dev_groups,
	.match		= platform_match,
	.uevent		= platform_uevent,
	.pm		= &platform_dev_pm_ops,
};
```
其中需要重点关注的是match()成员函数，正是此成员函数确定了platform_device和platform_driver之间是如何进行匹配，其定义在driver/base/platform.c中，如下代码所示：
```c
static int platform_match(struct device *dev, struct device_driver *drv)
{
	struct platform_device *pdev = to_platform_device(dev);
	struct platform_driver *pdrv = to_platform_driver(drv);

	/* When driver_override is set, only bind to the matching driver */
	if (pdev->driver_override)
		return !strcmp(pdev->driver_override, drv->name);

	/* Attempt an OF style match first */
	if (of_driver_match_device(dev, drv))
		return 1;

	/* Then try ACPI style match */
	if (acpi_driver_match_device(dev, drv))
		return 1;

	/* Then try to match against the id table */
	if (pdrv->id_table)
		return platform_match_id(pdrv->id_table, pdev) != NULL;

	/* fall-back to driver name match */
	return (strcmp(pdev->name, drv->name) == 0);
}
```
从platform_match的代码中可以看出，匹配platform_device和platform_driver有4种可能性，一是基于设备树风格的匹配；二是基于ACPI风格的匹配；三是匹配ID表（即platform_device设备名是否出现在platform_driver的ID表内）；第四种是匹配platform_device设备名和驱动的名字。

对于Linux 2.6内核的ARM平台而言，对platform_device的定义通常在BSP的板文件中实现，在板文件中，将platform_device归纳为一个数组，最终通过platform_add_devices()函数统一注册。platform_add_devices()函数可以将平台设备添加到系统中，这个函数定义在driver/base/platform.c中，它的实现为：
```c
int platform_add_devices(struct platform_device **devs, int num)
{
	int i, ret = 0;

	for (i = 0; i < num; i++) {
		ret = platform_device_register(devs[i]);
		if (ret) {
			while (--i >= 0)
				platform_device_unregister(devs[i]);
			break;
		}
	}

	return ret;
}
```
该函数的第一个参数为平台设备数组的指针，第二个参数为平台设备的数量，它内部调用了platform_device_register（）函数以注册单个的平台设备。

**但是在Linux 3.x之后，ARM Linux不太喜欢人们以编码的形式去填写platform_device和注册，而倾向于根据<font color=red>设备树</font>中的内容自动展开platform_device。**

# 二. platform设备驱动模板
```c
#include <linux/platform_device.h>
...


static int probe(struct platform_device *pdev)
{
	...
}
static int remove)(struct platform_device *pdev)
{
	...
}

static struct platform_driver xxx_driver = {
	.driver = {
		.name = "xxx",
		.onwer = THIS_MODULE,
		...
	},
	.probe = xxx_probe,
	.remove = xxx_remove,
	...
};
module_platform_driver(xxx_driver);
```
>模板中的module_platform_driver()是一个宏，其定义的模块加载和卸载函数仅仅通过platform_driver_register()、platform_driver_unregister()函数进行platform_driver的注册与注销，而原先注册和注销字符设备的工作已经被移交到platform_driver的probe()和remove()成员函数中。注册完驱动之后会在/sys/bus/platform/drivers中生成相应子目录。

**其余部分的代码与一般驱动一样，这里省略掉了**

在Linux2.6的内核中，还需要在板文件arch/arm/mach-<soc名>/mach-<板名>.c）中添加相应的代码，用来定义platform_device，
```c
static struct platform_device xxx_device = {
	.name = "xxx", /* 必须与platform_driver中一致 */
	.id = -1,
	...
};
```
并最终通过类似于platform_add_devices()的函数把这个platform_device注册进系统。一般板文件中定义了相应的platform_devices数组，然后统一对这个数组进行注册，因此也只需要把定义的platform_device添加到这个数组里面就可以了。  

------------------------------------------------------------

platform_device结构体中的resource，它们描述了platform_device的资源，resource结构体其定义在include/linux/ioport.h中，如下：
```c
struct resource {
	resource_size_t start;
	resource_size_t end;
	const char *name;
	unsigned long flags;
	struct resource *parent, *sibling, *child;
};
```
对于resource结构体,我们通常关心start、end和flags这3个字段，它们分别标明了资源的开始值、结束值和类型，flags可以为IORESOURCE_IO、IORESOURCE_MEM、IORESOURCE_IRQ、IORE-SOURCE_DMA等。start、end的含义会随着flags而变更，如当flags为IORESOURCE_MEM时，start、end分别表示该platform_device占据的内存的开始地址和结束地址；当flags为IORESOURCE_IRQ时，start、end分别表示该platform_device使用的中断号的开始值和结束值，如果只使用了1个中断号，开始和结束值相同。对于同种类型的资源而言，可以有多份，例如说某设备占据了两个内存区域，则可以定义两个IORESOURCE_MEM资源。  
对resource的定义也通常在板文件中进行(Linux2.6)，而在具体的设备驱动中通过platform_get_resource()这样的API来获取，此API定义在include/base/platform.c中，原型为：
```c
struct resource *platform_get_resource(struct platform_device *dev,
				       unsigned int type, unsigned int num)
{
	int i;

	for (i = 0; i < dev->num_resources; i++) {
		struct resource *r = &dev->resource[i];

		if (type == resource_type(r) && num-- == 0)
			return r;
	}
	return NULL;
}
```
对于IRQ而言，platform_get_resource()还有一个进行了封装的变体platform_get_irq()，其原型为：
```c
int platform_get_irq(struct platform_device *dev, unsigned int num)
{
#ifdef CONFIG_SPARC
	/* sparc does not have irqs represented as IORESOURCE_IRQ resources */
	if (!dev || num >= dev->archdata.num_irqs)
		return -ENXIO;
	return dev->archdata.irqs[num];
#else
	struct resource *r;
	if (IS_ENABLED(CONFIG_OF_IRQ) && dev->dev.of_node) {
		int ret;

		ret = of_irq_get(dev->dev.of_node, num);
		if (ret >= 0 || ret == -EPROBE_DEFER)
			return ret;
	}

	r = platform_get_resource(dev, IORESOURCE_IRQ, num);

	return r ? r->start : -ENXIO;
#endif
}
```
可以看到，其实质是调用了platform_get_resource(dev, IORESOURCE_IRQ, num);  
在内核中这种便捷封装有很多，在编写驱动的时候可以留意一下，以便使代码更简洁。

设备除了可以在BSP中定义资源以外，还可以附加一些数据信息，因为对设备的硬件描述除了中断、内存等标准资源以外，可能还会有一些配置信息，而这些配置信息也依赖于板，不适宜直接放置在设备驱动上。因此，platform也提供了platform_data的支持，platform_data的形式是由每个驱动自定义的，如对于DM9000网卡而言，platform_data为一个dm9000_plat_data结构体，完成定义后，就可以将MAC地址、总线宽度、板上有无EEPROM信息等放入platform_data中，具体代码如下：
```c
static struct dm9000_plat_data mini6410_dm9k_pdata = {
	.flags		= (DM9000_PLATF_16BITONLY | DM9000_PLATF_NO_EEPROM),
};

static struct platform_device mini6410_device_eth = {
	.name		= "dm9000",
	.id		= -1,
	.num_resources	= ARRAY_SIZE(mini6410_dm9k_resource),
	.resource	= mini6410_dm9k_resource,
	.dev		= {
		.platform_data	= &mini6410_dm9k_pdata,
	},
};
```
而在DM9000网卡的驱动drivers/net/ethernet/davicom/dm9000.c的probe()中，通过如下方式就拿到了platform_data：
```c
	struct dm9000_plat_data *pdata = dev_get_platdata(&pdev->dev);
```
其中，pdev为platform_device的指针。
>由以上分析可知，在设备驱动中引入platform的概念至少有如下好处。  
1）使得设备被挂接在一个总线上，符合Linux 2.6以后内核的设备模型。其结果是使配套的sysfs节点、设备电源管理都成为可能。  
2）隔离BSP和驱动。在BSP中定义platform设备和设备使用的资源、设备的具体配置信息，而在驱动中，只需要通过通用API去获取资源和数据，做到了板相关代码和驱动代码的分离，使得驱动具有更好的可扩展性和跨平台性。  
3）让一个驱动支持多个设备实例。譬如DM9000的驱动只有一份，但是我们可以在板级添加多份DM9000的platform_device，它们都可以与唯一的驱动匹配。

**以下是重点**

在Linux 3.x之后的内核中，DM9000驱动实际上已经可以通过设备树的方法被枚举，可以参见补丁net：dm9000：Allow instantiation using device tree（内核commit的ID是0b8bf1ba）。  
具体代码可以参见*driver/net/ethernet/davicom/dm9000.c*，从中可以看到设备树相关的代码  
改为设备树后，在板上添加DM9000网卡的动作就变成了简单地修改dts文件，如arch/arm/boot/dts/s3c6410-mini6410.dts中就有这样的代码：
```
	srom-cs1@18000000 {
		compatible = "simple-bus";
		#address-cells = <1>;
		#size-cells = <1>;
		reg = <0x18000000 0x8000000>;
		ranges;

		ethernet@18000000 {
			compatible = "davicom,dm9000";
			reg = <0x18000000 0x2 0x18000004 0x2>;
			interrupt-parent = <&gpn>;
			interrupts = <7 IRQ_TYPE_LEVEL_HIGH>;
			davicom,no-eeprom;
		};
	};
```
关于设备树情况下驱动与设备的匹配，以及驱动如何获取平台属性的更详细细节，将在后面介绍。