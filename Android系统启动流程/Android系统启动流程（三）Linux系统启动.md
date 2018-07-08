##1 Linux系统启动主要分为三个阶段
> * Linux系统启动主要分为三个阶段
> * 1.第一个阶段是自解压过程
> * 2.第二个是设置ARM处理器的工作模式、设置一级页表等
> * 3.第三个阶段主要是C代码，包括Android的初始化的全部工作。
##2 自解压过程
> * BootLoader完成系统的引导以后并将Linux内核调入内核之后，调用do_bootm_linux()，这个函数将跳转到kernel的其实位置。
> * 如果kernel没有被压缩，就可以启动了。如果kernel被压缩过，就要进行压缩了，在压缩过的kernel头部有解压缩程序。
> * 压缩过的kernel入口第一个文件源位置正在arch/arm/boot/compressed/head.S。它将调用函数decompress_kernel()函数，这个函数在文件的arch/arm/boot/compressed/misc.c中，decompress_kernel()又调用proc_decomp_setup()，arch_decomp_setup()进行设置，然后打印gunzip()将内核放于指定的位置。


> * 内核压缩和解压缩代码都在目录kernel/arch/boot/compressed
> * 编译完成后将产生head.o、misc.o、piggy.gzip.o、vmlinux、decompress.o这几个文件


> * head.o：是内核的头部文件，负责初始设置
> * misc.o：将主要负责内核的解压工作，它在head.o之后
> * piggy.gzip.o：是一个中间文件，其实是一个压缩的内核(kernel/vmlinux)，只不过没有和初始化文件及解压缩文件链接而已；
> * vmlinux：是没有(zImage是压缩过的内核)压缩过的内核，就是由piggy.gzip.o、head.o、misc.o组成的
> * decompress.o是未支持更多的压缩格式而新引入的。

**解压缩的过程(函数decompress_kernel的实现功能)**
> * 解压缩的代码位置与kernel/lib/inflate.c，inflate.c是从gzip源程序中分离出来的，包含一些对全局数据的直接引用，在使用时需要直接嵌入到代码中。
> * gzip压缩文件时总是在前32K字节的解压缩缓冲区，它定义为window[WSIZE]。
> * inflate.c使用get_byte()读取输入文件，它被定义成 宏 来提高效率。
> * 输入缓冲区指针必须定位inptr，inflate.c中对之有减量操作。inflate.c调用flush_window()来输出window缓冲区的解压出的字节串，每次输出长度用outcnt变量表示。
> * 在flushwindow()中，还必须对输出字节串计算CRC并且刷新crc变量。在调用gunzip()开始解压之前，调用makecrc()初始化CRC计算表。最后gunzip()返回0表示解压成功。

##3 Linux初始化
Linux初始化又分为三个阶段
###3.1 第一阶段
本阶段就是上面说的到的内核解压缩完成后的阶段。
> * 该部分的代码实现在arch/arm/kernel的 head.S中，该文件的汇编代码通过查找处理内和类型的机器码类型**调用相应的初始化函数**，再**建立页表**，最后跳转到**start_kernel()函数开始内核的初始化工作**。
> * **检查处理器**是汇编子函数__lookup_processor_type中完成的
> * **检查机器码类型**是汇编函数__lookup_machine_type中完成
> * 当检测处理器类型和机器码类型结束后，将调用__create_page_tables子函数来**建立页表**，它所要做的工作就是将RAM地址开始的1M空间物理地址映射到0xC0000000开始的虚拟地址处。
> * 对本项目的开发板DM3730而言，RAM挂接到物理地址0x80000000处，当调用__create_page_tables结束后0x80000000 ～ 0x80100000物理地址将映射到0xC0000000～0xC0100000虚拟地址处。
> * 当所有的初始化结束之后，使用如下代码来跳到C程序的入口函数**start_kernel()**处，开始之后的内核初始化共工作。


Head.S核心代码如下
```
ENTRY(stext)
setmode PSR_F_BIT | PSR_I_BIT | SVC_MODE, r9 @设置SVC模式关中断
      mrc p15, 0, r9, c0, c0        @ 获得处理器ID，存入r9寄存器
      bl    __lookup_processor_type        @ 返回值r5=procinfo r9=cpuid
      movs      r10, r5                       
 THUMB( it eq )        @ force fixup-able long branch encoding
      beq __error_p                   @如果返回值r5=0，则不支持当前处理器'
      bl    __lookup_machine_type         @ 调用函数，返回值r5=machinfo
      movs      r8, r5            @ 如果返回值r5=0，则不支持当前机器（开发板）
THUMB( it   eq )             @ force fixup-able long branch encoding
      beq __error_a                   @ 机器码不匹配，转__error_a并打印错误信息
      bl    __vet_atags
#ifdef CONFIG_SMP_ON_UP    @ 如果是多核处理器进行相应设置
      bl    __fixup_smp
#endif
      bl    __create_page_tables  @最后开始创建页表
```
###3.2 第二阶段
> * 从start_kernel函数开始
> * Linux内核启动的第一个阶段是从start_kernel函数开始的。
> * start_kernel是所有Linux平台进入系统内核**初始化后的入口函数**，它主要完成**剩余与硬件平台的相关初始化工作**，在进行一些系列的与内核相关的初始后，**调用第一个用户进程——init进程并等待用户进程的执行**，这样整个Linux内核便启动完毕。
> * 该函数位于init/main.c文件中。
![5713484-fb928231e6856610.png](https://upload-images.jianshu.io/upload_images/5982616-49487ad8ca33db35.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

**这个函数内部的具体工作如下：**
> * 调用**setup_arch()函数**进行与体系结构相关的第一个初始化工作；对于不同的体系结构来说该函数有不同的定义。对于ARM平台而言，该函数定义在arch/arm/kernel/setup.c。它首先**通过检测出来的处理器类型**进行处理其内核的初始化，然后通过bootmem_init()函数根据系统定义的meminfo结构进行**内存结构的初始化**，最后调用 paging_init()开启MMU，创建内核页表，映射所有的物理内存和IO空间。
> * 创建异常向量表和初始化中断处理函数
> * 初始化系统核心进程调度器和时钟中断处理机制
> * 初始化串口控制台
> * 创建**初始化系统cache**，为各种内存调用机制提供缓存，包括动态内存分配，虚拟文件系统(VirtuaFile System)及页缓存。
> * **初始化内存管理**，检测内存大小及被内核占用的内存情况。
> * **初始化系统的进程间通信机制(IPC)**；当以上所有的初始化工作结束后，start_kernel()函数会**调用rest_init()函数**来进行最后的初始化，包括创建系统的第一个进程——init进程来结束内核的启动

> * Linux内核启动的最后一个阶段就是挂载根文件系统，因为启动第一个init进程，必须以根文件系统为载体。

###3.3 第三阶段
根文件系统至少包括以下目录：
> * /etc/：存储重要的配置文件
> * /bin/：存储常用且开机时必须用到的执行文件。
> * /sbin/：存储着开机过程中所需的系统执行文件。
> * /lib/：存储/bin/及/sbin/的执行文件所需要的链接库，以及Linux的内核模块
> * /dev/：存储设备文件 
上面五大目录必须存储在跟文件系统上，缺一不可。

**1、为什么以只读的方式**
以只读的方式挂载根文件系统，之所以采用只读的方式挂载根文件系统是因为：此时Linux内核仍在启动阶段，还不是很稳定，如果采用可读可写的方式挂载跟文件系统，万一Linux不小心宕机了，一来可能破坏根文件系统上的数据，再者Linux下次开机时得花上很长时间来检查并修复根文件系统。

**2、挂载根文件系统的目的：**
> * 安装适当的内核模块，以便驱动某些硬件设备或启动某些功能
> * 启动存储于文件系统中的init服务，以便让init服务接手后续的启动工作。

**3、执行init服务的顺序：**
Linxu内核启动的最后一个动作，就是从根文件系统上找出并执行init服务。Linux内核会依照下列的顺序寻找init服务：
> * 第一步检查 /sbin/是否有init服务
> * 第二步检查/etc/是否有init服务
> * 第三步检查/bin/是否有init服务
> * 如果都找不到你最后执行/bin/sh

**找到init服务**后，Linux会让init服务负责后续初始化系统使用环境的工作，**init启动后，就代表系统已经顺利地启动了Linux内核**。启动init服务时，**init服务会读取/etc/inittab文件**，根据/etc/inittab中的设置数据进行初始化系统环境工作。

**4、/etc/inittab：**
/ect/inittabl定义init服务在Linux启动过程中必须执行以下几个脚本：
> * /etc/rc.d/rc.sysinit
主要功能是设置系统的基本环境，当init服务执行rc.sysinit时，要依次完成下面一系列工作：
 * 启动udev
 * 设置内核参数：执行sysctl -p，以便从/etc/sysctl.conf设置内核参数
 * 设置**系统时间**：将硬件时间设置为系统时间
 * 启动交换内存空间：执行swpaon -a -e，以便根据、etc/fstab的设置启动所有的交互内存空间。
 * 检查并挂载所有文件系统：检查所有需要挂载的文件系统，以确保这些文件系统的完整性。检查完毕后可读可写的方式挂载文件系统。
 * 初始化硬件设备：Linux除了在启动内核时以静态驱动部分的硬件外，在执行rc.sysinit时，也会试着驱动剩余的硬件设备。
 * 初始化串行端口设备：Init服务会管理所有的串行端口设备，比如调制解调器、不断电系统、串行端口控制台等。Init服务则通过rc.sysinit来初始化Linux串行端口设备。当rc.sysinit发现Linux才能在这/etc/rc.serial时，才会执行/etc/rc.serial，借以初始化所有的串行端口设备。因此，你可以在/etc/rc.serial中定义如何初始化Linux所有的串行端口设备。
 * 清除过程的锁定文件与IPC文件
 * 建立用户接口
 * 建立虚拟控制台
> * /etc/rc.d/rc
> * /etc/rc.d/rc.local
这里简单说一下建立虚拟控制台
> * init会在若干个虚拟控制台中执行/bin/login，以便用户可以从虚拟控制台登录Linux。Linux默认在前6个虚拟控制台，也就tty1~tty6，执行/bin/login登录程序。当所有的初始化工作结束后，**cpu-idle()函数会被调用来使用系统处于闲置(idle)状态并等待用户程序的执行**。至此，整个Linux内核启动完毕

##参考
[Android启动流程——1序言、bootloader引导与Linux启动](https://www.jianshu.com/p/9f978d57c683)


