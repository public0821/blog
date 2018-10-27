# Linux下常见文件系统对比

本文将对Linux下常见的几种文件系统进行对比，包括ext2、ext3、ext4、XFS和Btrfs，希望能帮助大家更好的选择合适的文件系统。

>内容来自于网上找的资料以及自己的一些经验，能力有限，错误在所难免，仅供参考

## 历史

| 文件系统 | 创建者 | 创建时间 |最开始支持的平台|
| :--- |  :--- |  :--- | :--- |
|[ext2](https://en.wikipedia.org/wiki/Ext2)|[Rémy Card](https://en.wikipedia.org/wiki/R%C3%A9my_Card)|1993|Linux,[Hurd](https://en.wikipedia.org/wiki/GNU_Hurd)|
|[XFS](https://en.wikipedia.org/wiki/XFS)|[SGI](https://en.wikipedia.org/wiki/Silicon_Graphics)| 1994|[IRIX](https://en.wikipedia.org/wiki/IRIX), Linux, FreeBSD|
|[ext3](https://en.wikipedia.org/wiki/Ext3)|[Dr. Stephen C. Tweedie](https://en.wikipedia.org/wiki/Stephen_Tweedie)| 1999|Linux|
|[ZFS](https://en.wikipedia.org/wiki/ZFS)|Sun| 2004|Solaris|
|[ext4](https://en.wikipedia.org/wiki/Ext4)|众多开发者| 2006|Linux|
|[Btrfs](https://en.wikipedia.org/wiki/Btrfs)|Oracle| 2007|Linux|

从创建时间可以看出他们所处的不同时代，因为Btrfs的实现借鉴自ZFS，所以这里也将ZFS列出来作为参考。

## 大小限制

| 文件系统 | 最大文件名长度 | 最大文件大小 |最大分区大小|
| :--- |  :--- |  :--- | :--- |
|ext2|255 bytes |2 TB   |16 TB|
|ext3|255 bytes|    2 TB    |16 TB|
|ext4|255 bytes|    16 TB|  1 EB|
|XFS|255 bytes  |8 EB|  8 EB|
|Btrfs|255 bytes|   16 EB|  16 EB|

最大文件和分区大小受格式化分区时所采用的块大小（block size）所影响，块越大，所支持的最大文件和分区越大，也越可能浪费磁盘空间，上表列出的数据基于4K的块大小。

## 代码规模
从代码规模可以看出文件系统的功能丰富程度以及复杂度，下面列出的数据来自于kernel-4.1-rc8，只是简单的用wc -l来统计，没有过滤空行、注释等。

| 文件系统 | 源文件(.c) | 头文件(.h) |
| :--- |  :--- |  :--- |
|ext2 |8363 |1016|
|ext3|16496 |1567|
|ext4|44650|    4522|
|XFS|89605  |15091|
|Btrfs|105254   |7933|

* Btrfs还在快速的开发过程中，代码行数可能还有比较大的变化
* XFS和Btrfs都使用了B-tree

## ext2
ext的优点是比较简单，文件比较少时性能较好，比较适合文件少的场景，主要缺点如下

* inode的数量是固定不变的，在格式化分区的时候可以指定inode和数据块所占空间的比例，但一旦格式化好，后续就没法再改变了
* 当块大小为4K时，单个文件大小不能超过2TB，分区大小不能超过16TB（目前硬盘大小一般都只有几TB，所以也不是什么大问题，）
* 一个目录下最多只能有32000个子目录
* 由于目录里面存储的文件和子目录都是以线性方式来组织的，所以遍历目录效率不高，尤其当目录下文件个数达到10K以上规模的时候，速度会明显的变慢
* 当底层的磁盘分区空间变大时（使用LVM时很常见），ext2没法动态的扩展来使用增加的空间
* 没有日志（Journal）功能，所以数据的安全性不高

## ext3
ext3在ext2的基础上实现了下面几个功能，其它的都保持不变，即ext2的缺点ext3也有

* 支持日志（Journal）功能，数据的安全性较ext2有很大的提高
* 当底层的分区空间变大时，ext3可以自动扩展来使用增加的空间
* 使用HTree来组织目录里面的文件和子目录，使目录下的文件和子目录数不再受性能限制（数量超过10K也不会有性能问题）

## ext4
ext4借鉴了当前成熟的一些文件系统技术，在ext3上增加了一些功能，并且对性能做了一些改进，主要变化如下

* 当块大小为4K时，支持的最大文件和最大分区大小分别达到了16TB和1EB
* 不再受32000个子目录数的限制，支持不限数量的子目录个数
* 支持Extents，提高了大文件的操作性能
* 内部实现上支持一次分配多个数据块，较ext3的性能有所提高
* 支持延时分配（即支持fallocate函数）（fallocate是libc的函数，在不支持该功能的文件系统上，libc会创建一个占用磁盘空间文件）
* 支持在线快速扫描
* 支持在线碎片整理（单个文件或者整个分区）
* 日志（Journal）支持校验码（checksum），数据的安全性进一步提高
* 支持无日志（No Journaling）模式（ext3不支持该功能），这样就和ext2一样，消除了写日志对性能的影响
* 支持纳秒级的时间戳
* 记录了文件的创建时间，由于相关的应用层工具还不支持，所以只能通过debug的方式看到文件的创建时间

这里是一个查看文件/etc/fstab创建时间的例子（文件存在/dev/sda1分区上）：
```bash
dev@ubuntu:~$ ls -i /etc/fstab
10747906 /etc/fstab
dev@ubuntu:~$ sudo debugfs -R 'stat <10747906>' /dev/sda1
Inode: 10747906   Type: regular    Mode:  0644   Flags: 0x80000
Links: 1   Blockcount: 8
ctime: 0x5546dc54:6e6bc80c -- Sun May  3 22:41:24 2015
 atime: 0x55d1b014:8bcf7b44 -- Mon Aug 17 05:57:40 2015
 mtime: 0x5546dc54:6e6bc80c -- Sun May  3 22:41:24 2015
crtime: 0x5546dc54:6e6bc80c -- Sun May  3 22:41:24 2015
Size of extra inode fields: 28
EXTENTS: (0):46712815
```


**Extents：** 在最开始的ext2文件系统中，数据块都是一个一个单独管理的，inode中存有指向数据块的指针，文件占用了多少个数据块，inode里面就有多少个指针（多级），想象一下一个1G的文件，4K的块大小，那么需要(1024 * 1024)/4=262144个数据块，即需要262144个指针，创建文件的时候需要初始化这些指针，删除文件的时候需要回收这些指针，影响性能。现代的文件系统都支持Extents的功能，简单点说，Extent就是数据块的集合，以前一次分配一个数据块，现在可以一次分配一个Extent，里面包含很多数据块，同时inode里面只需要分配指向Extent的指针就可以了，从而大大减少了指针的数量和层级，提高了大文件操作的性能。

**inode数量固定：** 在ext2/3/4系列的文件系统中，inode的数量都是固定的，坏处是如果存很多小文件的话，有可能造成inode被用光，但磁盘还有很多剩余空间无法被使用的情况，不过它也有一个好处，就是一旦磁盘损坏，恢复起来要相对简单些，因为数据在磁盘上布局相对要固定简单。

## xfs
和ext4相比，xfs不支持下面这些功能

* 不支持日志（Journal）校验码
* 不支持无日志（No Journaling）模式
* 不支持文件创建时间
* 不支持数据日志（data journal），只有元数据日志（metadata journal）

但xfs有下面这些特性

* 支持的最大文件和分区都达到了8EB
* inode动态分配，从而不受inode数量的限制，再也不用担心存储大量小文件导致inode不够用的问题了。
* [更大的xattr(extended attributes)](https://en.wikipedia.org/wiki/XFS#Extended_attributes)空间，ext2/3/4及btrfs都限制xattr的长度不能超过一个块（一般是4K），而xfs可以达到64K
* 内部采用Allocation groups机制，各个group之间没有依赖，支持并发操作，在多核环境的某些场景下性能表现不错
* 提供了原生的dump和restore工具，并且支持在线dump

## btrfs
btrfs是一个和ZFS类似的文件系统，支持的功能非常多，据说将来会替换ext4成为Linux下的默认文件系统。这里列举一些重要的功能

* 支持的最大文件和分区达到了16EB
* 支持COW（copy on write）
* 针对小文件和SSD做了优化
* inode动态分配
* 支持子分区（Subvolumes），子分区可以单独挂载
* 支持元数据和数据的校验（crc32）
* 支持压缩，去重
* 支持多个磁盘和分区，可动态扩展
* 支持LVM,RAID的功能（有了btrfs，就不再需要lvm和软raid了）
* 增量备份和恢复
* 支持快照
* 将ext2/3/4转换成btrfs（反过来不行）

btrfs最大的缺点就是由于其COW的实现方式，导致碎片化问题比较严重，不太适合频繁写的场景，比如数据库、虚拟机的磁盘文件等。不过大部分场合不需要担心，btrfs有在线的碎片整理工具。

## 如何选择
下表仅供参考

|文件系统 |适用场景 |原因|
| :--- |  :--- |  :--- |
|ext2| U盘 |U盘一般不会存很多文件，且U盘的文件在电脑上有备份，安全性要求没那么高，由于ext2不写日志（journal），所以写U盘性能比较好。当然由于ext2的兼容性没有fat好，目前大多数U盘格式还是用fat|
|ext3 |对稳定性要求高的地方  |有了ext4后，好像没什么原因还要用ext3，ext4现在的问题是出来时间不长，还需要一段时间变稳定|
|ext4 |小文件较少 |ext系列的文件系统都不支持inode动态分配，所以如果有大量小文件需要存储的话，不建议用ext4|
|xfs |小文件多或者需要大的xttr空间，如[openstack swift](https://docs.openstack.org/developer/swift/)将数据文件的元数据放在了xttr里面 |xfs支持inode动态分配，所以不存在inode不够的情况，并且xttr的最大长度可以达到64K|
|btrfs |没有频繁的写操作，且需要btrfs的一些特性 |btrfs虽然还不稳定，但支持众多的功能，如果你需要这些功能，且不会频繁的写文件，那么选择btrfs|

另外，ext系列文件系统内部结构相对简单一些，出问题后恢复相对容易。

## 结束语
本篇没有比较它们的性能，在通常情况下，他们之间没有太大的性能差别，只有在特定的场景下，才能看出区别，如果对性能比较敏感，建议根据自己的使用场景来测试不同的文件系统，然后根据结果来选择。
