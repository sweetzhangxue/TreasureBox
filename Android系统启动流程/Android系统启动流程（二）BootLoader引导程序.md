## 一、Bootloader的定义和种类
> * BootLoader是在操作系统运行之前运行的一段程序，它可以将系统的软硬件环境带到一个合适的状态，为运行操作系统做好准备。
> * **终极目标就是把操作系统拉起来运行**
在嵌入式系统世界里存在各种各样的BootLoader，种类划分也有多种方式

###1.1**区分Bootloader和Monitor**
> * Bootloader只是引导操作系统运行起来的代码
> * Monitor另外还提供很多命令接口，可以进行调试、读写内存、配置环境变量等。
> * 在开始过程中Monitor提供了很好的调试功能，不过在开始结束之后，可以将其设置成一个Bootloader。

![TIM截图20180706154515.png](https://upload-images.jianshu.io/upload_images/5982616-c8984207950bb991.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
> * x86：x86的工作站和服务器上一般使用LILO和GRUB
> * ARM：最早由为ARM720处理器开发板所做的固件，又有armboot，StrongARM平台的blob，还有S3C2410处理器开发板上的vivi等。现在armboot已并入U-Boot，所以U-Boot也支持ARM/XSALE平台。U-Boot已经成为ARM平台事实上的标准Bootloader
> * PowerPC：最早使用于ppcboot，不过现在大多数直接使用U-boot。
> * MIPS：最早都是MIPS开发商自己写的bootloader，不过现在U-boot也支持MIPS架构。
> * M68K：Redboot能够支持m68k系列的系统。

###1.2 **分区：ARM、处理器、CPU**
> * ARM本身是一个公司的名称，从技术的角度来看，它又是一种**微处理器内核的架构**。
> * 处理器一种统称，可以指具体的CPU芯片，比如intel i7处理器，苹果的A11处理器等。处理器内部一般包含CPU、片上内存、片上外设接口等不同的硬件逻辑。
> * CPU是处理器内部的中央处理单元的缩写，CPU可以按照类型分为短指令集架构和长指令集架构两大类，**ARM属于短指令集架构的一种**

##二 ARM特定平台的BootLoader
> * 对于ARM处理器，当复位完毕后，处理器首先执行其片上ROM中的一小块程序。这块ROM的大小一般只有几KB，该段程序就是Bootloader程序
> * 这段程序执行时会根据处理器上一些特定的引脚的高低电平状态，**选择从何种物理接口上装载用户程序**，比如UBS接口、串口、SD卡、并口Flash等。
> * 多数基于ARM的实际硬件系统，会从并口NAND Flash 芯片中的 0x00000000地址处装载程序。
> * 对于Android而言，该地址中的程序还不是Android程序，而是一个叫做**uboot或者fastboot的程序**，其作用就是**初始化硬件设备**，比如网口、SDRAM、RS232等，并提供一些调试功能，比如像NAND Flash写入新的数据，这可用于开发过程中的内核烧写、等级等。
> * 当uboot(fastboot)被装载后便开始运行，它一般会先检测用户是否按下某些特别按键，这些特别按键是uboot在编译时预先被约定好的，用于进入调试模式。
> * 如果用户没有按这些特别的按键，则uboot会**从NAND Flash中装载Linux内核**，装载的地址是在编译uboot时预先约定好的。

**上电之后到U-boot的流程**
![5713484-5b860a6e721201d1.png](https://upload-images.jianshu.io/upload_images/5982616-c10d91128bbcc0ae.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


##三 U-boot启动流程分析
> * 手机系统不像其他的嵌入式系统，它还需要在启动的过程中关心**CP的启动**，这个时候就涉及到CP的image和唤醒时刻，而一般的嵌入式系统的uboot只负责引导OS内核。

**暂不关心CP的启动，而主要关心AP**
> * U-boot的启动过程大致上可以分为两个阶段：
> * 第一阶段：汇编代码
U-boot的第一条指令从cpu/armXXX/start.S文件开始
> * 第二阶段：C代码
从文件/lib_arm/board.c的start_armboot()函数开始。


##参考
[Android启动流程——1序言、bootloader引导与Linux启动](https://www.jianshu.com/p/9f978d57c683)


