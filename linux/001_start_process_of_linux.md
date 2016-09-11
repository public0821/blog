#Linux的启动过程

本文将简单介绍一下Linux的启动过程，希望对那些安装Linux的过程中遇到了问题的朋友有些帮助

>**声明：** 本人没用过UEFI模式和GPT分区格式，所有关于这两部分的内容都是网络上找的资料，仅供参考。

##典型启动顺序
1. 计算机通电后，CPU开始从一个固定的地址加载代码并开始执行，这个地址就是BIOS的驱动程序所在的位置，于是BIOS的驱动开始执行。

2. BIOS驱动首先进行一些自检工作，然后根据配置的启动顺序，依次尝试加载启动程序。比如配置的启动顺序是CD->网卡01->USB->硬盘。 BIOS 将先检查是否能从CD启动，如果不行，接着试着从网卡启动，再试USB盘，最后再试硬盘。

3. CD，U盘和硬盘的启动都是一样的，对BIOS来说，它们都是块设备，BIOS通过硬件访问接口直接访问这些块设备（如通过IDE访问硬盘），加载固定位置（第一个扇区）的内容到内存，然后跳转到那个内存的位置开始执行，这里固定位置所存放的就是Bootloader的代码，从这个时间点开始，启动的工作就由BIOS交接到了Bootloader手中了。现在Linux下常用的Bootloader是[GRUB2](https://www.gnu.org/software/grub/)，当然开源的Bootloader有[很多种](https://en.wikipedia.org/wiki/Comparison_of_boot_loaders)，并且各有各的特点.

4. 从网卡启动稍微有所不同，当然前提条件是网卡支持PXE启动。 下面是大概的步骤
    1. 从网卡中加载PXE firmware到内存并执行，里面主要包含一个很小的网络驱动和TFTP client的实现
    
    2. 发送UDP广播到当前局域网，向DHCP服务器要IP和NBP(Network Boot Program)的地址
    
    3. DHCP服务器收到广播后，会发送应答，里面包含分配给请求机器的IP以及NBP的所在位置
    
    4. 将分配的IP应用到网卡上，然后根据收到的NBP的地址，用TFTP协议到相应的服务器上取相应的NBP文件（取文件的过程不再是广播，而是点对点的文件传输过程，所以当前网卡必须要有IP）
    
    6. 开始执行取到的NBP（Linux一般使用[pxelinux](http://www.syslinux.org/wiki/index.php?title=PXELINUX)作为NBP）
    
>从上面的过程可以看出，一个PXE服务器至少包含一个DHCP server和一个TFTP server。

##以硬盘启动及GRUB2为例，接着介绍Linux的启动过程
1. BIOS加载硬盘[MBR](https://en.wikipedia.org/wiki/Master_boot_record)中的[GRUB](https://zh.wikipedia.org/wiki/GNU_GRUB)后，启动过程就被GRUB2接管

2. 由于MBR里面空间很小，GRUB2只能放部分代码到里面，所以它采用了好几级的结构来加载自己，详情请点[这里](https://en.wikipedia.org/wiki/GNU_GRUB#Booting)，总之，最后GRUB2会加载/boot/grub/下的驱动到内存中。   

3. GRUB2加载内核和initrd image，并启动内核。GRUB2和内核之间的协议请参考[i386/boot.txt](https://www.kernel.org/pub/linux/kernel/people/marcelo/linux-2.4/Documentation/i386/boot.txt)。

4. 内核接管整个系统后，加载/sbin/init并创建第一个用户态的进程
    
5. init进程开始调用一系列的脚本来创建很多子进程，这些子进程负责初始化整个系统


##注意事项：

###GRUB2
GRUB2需要加载/boot下的grub模块才能工作，所以格式化Linux分区一定要注意，如果不小心格式化了/boot所在的分区，会导致GRUB2用不了，从而启动不了任何系统。
GRUB2同时需要加载硬盘上的Linux内核文件，所以它也需要有文件系统的驱动，当然它只需要读取文件，所以驱动很小。GRUB2已经支持所有的常见文件系统，并且完全支持LVM和RAID。

参考：

* [GRUB2: Differences from previous versions](http://www.gnu.org/software/grub/manual/grub.html#Changes-from-GRUB-Legacy)
* [GRUB2: features](http://www.gnu.org/software/grub/manual/grub.html#Features)

###[BIOS](https://en.wikipedia.org/wiki/BIOS) VS [UEFI](https://en.wikipedia.org/wiki/Unified_Extensible_Firmware_Interface)
UEFI可以简单理解为新一代的BIOS，支持更多新的功能，当然它也向下兼容BIOS，现在新的主板都支持UEFI，只是我们BIOS叫习惯了，所以就算主板已经支持新的UEFI，我们还是把它当BIOS用。UEFI的优点请参考[这里](https://en.wikipedia.org/wiki/Unified_Extensible_Firmware_Interface#Advantages)。

BIOS和UEFI两者启动系统的方式不一样，BIOS是读取硬盘第一个扇区的MBR到内存中，然后将控制权交给MBR里的Bootloader。而UEFI是读取efi分区，如果efi分区存在且里面有启动程序的话，将控制权交给启动程序，否则和BIOS一样，读取硬盘第一个扇区的MBR到内存中，将控制权交给MBR里面的Bootloader。从这里可以看出：

* UEFI是兼容BIOS的，就是说就算主板支持UEFI，只要我们不用efi分区，主板还是按照原来BIOS的方式来启动系统
* 两者只能选其一，使用efi分区里面的启动程序，或者是MBR里面的Bootloader
    
那什么时候应该用UEFI呢？

* 如果这台机器原来没有任何系统，那可以完全不用关心是BIOS还是UEFI，因为就算BIOS模式，Linux也可以从GPT盘启动
* 如果机器上已经有了一个系统，那么就必须确保新安装的Linux和原有的系统采取同样的模式。

如何判断原系统的模式：

* [Windows 8](http://www.howtogeek.com/175649/what-you-need-to-know-about-using-uefi-instead-of-the-bios/)及以上版本默认采用UEFI模式, Windows 7默认用BIOS模式
* [Ubuntu](https://help.ubuntu.com/community/UEFI#Identifying_if_an_Ubuntu_has_been_installed_in_UEFI_mode)

如何以UEFI模式安装： [Ubuntu](https://help.ubuntu.com/community/UEFI)


参考：

* [What is the difference in “Boot with BIOS” and “Boot with UEFI”](http://superuser.com/questions/496026/what-is-the-difference-in-boot-with-bios-and-boot-with-uefi)
* [Learn How UEFI Will Replace Your PC’s BIOS](http://www.howtogeek.com/56958/)

###MBR VS [GPT](https://en.wikipedia.org/wiki/GUID_Partition_Table)
MBR格式硬盘的布局
```
    ------------------------------------------------------------------
    |   |         |         |        |-------------------------------|
    |MBR| 主分区1  | 主分区2 | 主分区3 | 扩展 |逻辑分区1|...|逻辑分区n   |
    |   |         |         |        |-------------------------------|
    ------------------------------------------------------------------
                                        ↓ 
    扩展分区是一个特殊的主分区，分区最前面包含所有逻辑分区的描述，包含大小，位置等
```

* 由于留给MBR的空间太小，所以MBR格式的硬盘只能支持四个分区，就是我们常说的四个主分区。如果想把磁盘分成大于4个分区，就需要将其中的一个或者多个分区设置成扩展分区，然后在扩展分区里面划分逻辑分区。
* 对Linux而言，可以安装在主分区和逻辑分区里面，所以怎么划分硬盘都没关系。但对于Windows而言，由于只支持安装在主分区里面，所以必须至少有一个主分区，如果我们安装Linux时不小心将磁盘全部划分成逻辑分区，则以后要安装Windows就比较麻烦，需要重新划分磁盘分区格式。
* 同样由于留给MBR的空间太小，它所能表述的磁盘空间有限，只能支持小于2T的硬盘。

GPT主要用来替换MBR，并且配合UEFI使用。 在Windows和OS X上，只支持通过UEFI方式启动GPT硬盘，而FreeBSD，Linux依然支持BIOS模式启动GPT硬盘。

GPT的主要优点：

* 支持几乎无限制的磁盘分区个数，再也不需要主分区、扩展分区和逻辑分区这些概念了
* 支持超过2T的硬盘
* 分区数据在磁盘的不同位置存有多份，且有CRC校验码，所以更安全

参考：

* [What’s the Difference Between GPT and MBR When Partitioning a Drive?](http://www.howtogeek.com/193669/whats-the-difference-between-gpt-and-mbr-when-partitioning-a-drive/)

###内核参数和initrd image

下面是一个GRUB2配置的例子
```
    kernel /boot/vmlinuz-2.6.9-1.667 ro root=/dev/hda5 quiet
    initrd /boot/initrd-2.6.9-1.667.img
```
 当GRUB2加载完Linux内核（/boot/vmlinuz-2.6.9-1.667）后，将这里的“ro root=/dev/hda5 quiet”做为参数传给Linux内核，然后将控制权交给Linux内核。Linux支持的内核参数请点[这里](https://www.kernel.org/doc/Documentation/kernel-parameters.txt)，其中一个重要的参数是"init"

```
  'init=...'
      指定init程序的位置，Linux内核初始化完成后，将运行该位置所指定的程序    ，
      并将该进程作为第一个用户态进程，设置其进程ID为1
      如果没有指定这个参数，或者这个参数指定的位置不存在，
      Linux内核将依次搜索/sbin/init, /etc/init, /bin/init, /bin/sh这些路径，
      如果都不存在，Linux将启动失败。
    　　这里指定的init程序可以是可执行文件，软链接，也可以是脚本。

```

####initrd image是干嘛的呢？

我们都知道Linux内核模块的概念，比方说Linux支持N种不同的文件系统，Ext2/3/4，XFS, Btrfs等等，那需要把所有的这些文件系统驱动都编译进内核吗？当然不需要，因为这样做会导致内核太大，运行时占用太多的内存，取而代之，我们会把这些驱动编译成一个一个的内核模块，在需要用到的时候再把它们加载进内核，其它时间存放在磁盘上就好了。

现在有个问题，在GRUB将控制权交给Linux内核后，内核需要启动init程序，这个init程序是放在某个磁盘分区上的，这个磁盘分区用的是N个文件系统中的某一个，内核到哪里找这个文件系统的驱动呢？这个时候initrd image出场了，它里面包含了很多驱动模块，并且用的是内存文件系统，内存文件系统的驱动已经编译到内核中了，所以内核是可以直接访问initrd image的(老版本的initrd可能用的其它格式，但不管怎么样，肯定是被内核支持的格式)。当然initrd image里面不仅仅只包含文件系统的驱动，还有其它的很多文件，这个跟每个发行版有关，具体的内容可以参考相应的发行版。

####init
内核启动的第一个用户态进程init到底是个什么东东？其实它就是一个普通的程序，内核并没有对它做什么要求，只是别退出就好，init进程如果挂了的话，系统就崩溃了，至于init进程干些啥，启动其它的哪些进程，跟内核已经没有关系了，内核的任务就是管理硬件资源并调度这些用户态进程。我们也可以写一个我们自己的init程序放到那里，它也会正常的被内核启动起来。

除了在init进程里指定了handler的信号外，内核会帮init进程屏蔽掉其他所有信号，包括普通进程无法捕获和屏蔽的信号SIGKILL和SIGSTOP，这样可以防止其他进程不小心kill掉init进程导致系统挂掉。这是内核给用户态启动的第一个进程的特殊待遇。

init是用户态的第一个进程，所以非常重要，各个Linux发行版都用这个进程来创建很多子进程，然后让这些子进程来初始化用户态的环境，如mount各个分区，启动各个服务等，现在各个发行版主要采用这三种框架中的一种[sysvinit](http://savannah.nongnu.org/projects/sysvinit)，[upstart](http://upstart.ubuntu.com/)，[systemd](https://www.freedesktop.org/wiki/Software/systemd/)

简单点说，sysvinit出现最早，简单易用，但缺点是速度慢，比如有10个服务需要在开机时启动，那么sysvinit只能一个接一个的启动它们，即使他们之间没有任何关系，也不能并行的启动。于是出现了upstart，upstart基于事件驱动，可以让没有关系的服务并行的启动，这样可以加快开机速度。但是人们觉得还是不够快，于是出现了systemd，它可以通过一定的技术和技巧让有关系的服务也能并发的启动，当然导致的结果是systemd比较复杂。这里只提到了启动速度，当然还有其他方面的改进，详情请参考：

* [浅析 Linux 初始化 init 系统，第 1 部分: sysvinit](https://www.ibm.com/developerworks/cn/linux/1407_liuming_init1/)
* [浅析 Linux 初始化 init 系统，第 2 部分: UpStart](https://www.ibm.com/developerworks/cn/linux/1407_liuming_init2/)
* [浅析 Linux 初始化 init 系统，第 3 部分: Systemd](https://www.ibm.com/developerworks/cn/linux/1407_liuming_init3/)
