# Linux进程的内存使用情况

在linux下，使用top，ps等命令查看进程的内存使用情况时，经常看到VIRT，RES，SHR等，他们都代表什么意思呢？不同的大小对进程有什么影响呢？这篇文章将来聊一聊这个问题。阅读本篇前建议先阅读[Linux内存管理](004_memeory_management.md)，了解一些Linux下内存的基本概念，如什么是anonymous和file backed映射等。

## 查看进程所使用的内存
在进程的眼里，所有的内存都是虚拟内存，但是这些虚拟内存所对应的物理内存是多少呢？正如我们在[Linux内存管理](004_memeory_management.md)中所介绍的那样，并不是每块虚拟内存都有对应的物理内存，可能对应的数据在磁盘上的一个文件中，或者交换空间上的一块区域里。一个进程真正的物理内存使用情况只有内核知道，我们只能通过内核开放的一些接口来获取这些统计数据。

## top
先看看top的输出（top用到的数据来自于/proc/[pid]/statm），这里只是摘录了几条数据：
```
  PID USER      PR  NI    VIRT    RES    SHR S %CPU %MEM     TIME+ COMMAND
 2530 root      20   0       0      0      0 S  0.3  0.0   0:02.69 kworker/0:0
 2714 dev       20   0   41824   3700   3084 R  0.3  0.7   0:00.02 top
 3008 dev       20   0   22464   5124   3356 S  0.0  1.0   0:00.02 bash
```

VIRT：进程所使用的虚拟内存大小

RES：系统为虚拟内存分配的物理内存大小，包括file backed和anonymous内存，其中anonymous包含了进程自己分配和使用的内存，以及和别的进程通过mmap共享的内存；而file backed的内存就是指加载可执行文件和动态库所占的内存，以及通过private方式调用mmap映射文件所使用的内存（当在内存中修改了这部分数据且没有写回文件，那么这部分内存就变成了anonymous），这部分内存也可能跟别的进程共享。

SHR：RES的一部分，表示和别的进程共享的内存，包括通过mmap共享的内存和file backed的内存。当通过prive方式调用mmap映射一个文件时，如果没有修改文件的内容，那么那部分内容就是SHR的，一旦修改了文件内容且没有写回文件，那么这部分内存就是anonymous且非SHR的。

%MEM：等于RES/total*100%，这里total指总的物理内存大小。

>注意：由于SHR可能会被多个进程所共享，所以系统中所有进程的RES加起来可能会超过总的物理内存数量，由于同样的原因，所有进程的%MEM总和可能超过100%。

从上面的分析可以看出，VIRT的参考意义不大，它只能反应出程序的大小，而RES也不能完全的代表一个进程真正占用的内存空间，因为它里面还包含了SHR的部分，比如三个bash进程共享了一个libc动态库，那么libc所占用的内存算谁的呢？三个进程平分吗？如果启动一个bash占用了4M的RES，其中3M是libc占用的，由于三个进程都共享那3M的libc，那么启动3个bash实际占用的内存将是3*(4-3)+3=6M，但是如果单纯的按照RES来算的话，三个进程就用了12M的空间。所以理解RES和SHR这两个数据的含义对我们在评估一台服务器能跑多少个进程时尤其重要，不要一看到apache的进程占用了20M，就认为系统能跑的apache进程数就是总的物理内存数除以20M，其实这20M里面有可能有很大一部分是SHR的。

>注意：top命令输出中的RES和pmap输出中的RSS是一个东西。

## pmap
上面top命令只是给出了一个进程大概占用了多少的内存，而pmap能更详细的给出内存都是被谁占用了。pmap命令输出的内容来自于/proc/[pid]/maps和/proc/[pid]/smaps这两个文件，第一个文件包含了每段的一个大概描述，而后一个文件包含了更详细的信息。

这里用pmap看看当前bash的内存使用情况，：
```bash
#这里$$代表当前bash的进程ID，下面只显示了部分输出结果
dev@dev:~$ pmap  $$
2805:   bash
0000000000400000    976K r-x-- bash
00000000006f3000      4K r---- bash
00000000006f4000     36K rw--- bash
00000000006fd000     24K rw---   [ anon ]
0000000000be4000   1544K rw---   [ anon ]
......
00007f1fa0e9e000   2912K r---- locale-archive
00007f1fa1176000   1792K r-x-- libc-2.23.so
00007f1fa1336000   2044K ----- libc-2.23.so
00007f1fa1535000     16K r---- libc-2.23.so
00007f1fa1539000      8K rw--- libc-2.23.so
00007f1fa153b000     16K rw---   [ anon ]
......
00007f1fa196c000    152K r-x-- ld-2.23.so
00007f1fa1b7e000     28K r--s- gconv-modules.cache
00007f1fa1b85000     16K rw---   [ anon ]
00007f1fa1b8f000      8K rw---   [ anon ]
00007f1fa1b91000      4K r---- ld-2.23.so
00007f1fa1b92000      4K rw--- ld-2.23.so
00007f1fa1b93000      4K rw---   [ anon ]
00007ffde903a000    132K rw---   [ stack ]
00007ffde90e4000      8K r----   [ anon ]
00007ffde90e6000      8K r-x--   [ anon ]
ffffffffff600000      4K r-x--   [ anon ]
 total            22464K
```

这里第一列是内存的起始地址，第二列是mapping的地址大小，第三列是这段内存的访问权限，最后一列是mapping到的文件。这里的地址都是虚拟地址，大小也是虚拟地址大小。

这里的输出有很多的[ anon ]行，表示在磁盘上没有对应的文件，这些一般都是可执行文件或者动态库里的bss段。当然有对应文件的mapping也有可能是anonymous，比如文件的数据段。关于程序的数据段和bss段的介绍请参考[elf的相关资料](http://man7.org/linux/man-pages/man5/elf.5.html)。

上面可以看到bash、libc-2.23.so等文件出现了多行，但每行的权限不一样，这是因为每个动态库或者可执行文件里面都分很多段，有只能读和执行的代码段，有能读写的数据段，还有比如这一行“00007f1fa153b000     16K rw---   [ anon ]”，就是它上面一行libc-2.23.so的bss段。

[ stack ]表示进程用到的栈空间，而heap在这里看不到，因为pmap默认情况下不单独标记heap出来，由于heap是anonymous，所以从这里的大小可以推测出来，heap就是“0000000000be4000   1544K rw---   [ anon ]”。

其实从上面的结果根本看不出实际上每段占用了多少物理内存，要想看到RSS，需要使用-X参数，下面看看更详细的输出：
```bash
dev@dev:~$ pmap -X $$
2805:   bash
         Address Perm   Offset Device  Inode  Size  Rss  Pss Referenced Anonymous Shared_Hugetlb Private_Hugetlb Swap SwapPss Locked Mapping
        00400000 r-xp 00000000  fc:00 390914   976  888  526        888         0              0               0    0       0      0 bash
        006f3000 r--p 000f3000  fc:00 390914     4    4    4          4         4              0               0    0       0      0 bash
        006f4000 rw-p 000f4000  fc:00 390914    36   36   36         36        36              0               0    0       0      0 bash
        006fd000 rw-p 00000000  00:00      0    24   24   24         24        24              0               0    0       0      0
        00be4000 rw-p 00000000  00:00      0  1544 1544 1544       1544      1544              0               0    0       0      0 [heap]
    .....
    7f1fa0e9e000 r--p 00000000  fc:00 136340  2912  400   83        400         0              0               0    0       0      0 locale-archive
    7f1fa1176000 r-xp 00000000  fc:00 521726  1792 1512   54       1512         0              0               0    0       0      0 libc-2.23.so
    7f1fa1336000 ---p 001c0000  fc:00 521726  2044    0    0          0         0              0               0    0       0      0 libc-2.23.so
    7f1fa1535000 r--p 001bf000  fc:00 521726    16   16   16         16        16              0               0    0       0      0 libc-2.23.so
    7f1fa1539000 rw-p 001c3000  fc:00 521726     8    8    8          8         8              0               0    0       0      0 libc-2.23.so
    7f1fa153b000 rw-p 00000000  00:00      0    16   12   12         12        12              0               0    0       0      0
    ......
    7f1fa196c000 r-xp 00000000  fc:00 521702   152  144    4        144         0              0               0    0       0      0 ld-2.23.so
    7f1fa1b7e000 r--s 00000000  fc:00 132738    28   28    9         28         0              0               0    0       0      0 gconv-modules.cache
    7f1fa1b85000 rw-p 00000000  00:00      0    16   16   16         16        16              0               0    0       0      0
    7f1fa1b8f000 rw-p 00000000  00:00      0     8    8    8          8         8              0               0    0       0      0
    7f1fa1b91000 r--p 00025000  fc:00 521702     4    4    4          4         4              0               0    0       0      0 ld-2.23.so
    7f1fa1b92000 rw-p 00026000  fc:00 521702     4    4    4          4         4              0               0    0       0      0 ld-2.23.so
    7f1fa1b93000 rw-p 00000000  00:00      0     4    4    4          4         4              0               0    0       0      0
    7ffde903a000 rw-p 00000000  00:00      0   136   24   24         24        24              0               0    0       0      0 [stack]
    7ffde90e4000 r--p 00000000  00:00      0     8    0    0          0         0              0               0    0       0      0 [vvar]
    7ffde90e6000 r-xp 00000000  00:00      0     8    4    0          4         0              0               0    0       0      0 [vdso]
ffffffffff600000 r-xp 00000000  00:00      0     4    0    0          0         0              0               0    0       0      0 [vsyscall]
                                             ===== ==== ==== ========== ========= ============== =============== ==== ======= ======
                                             22468 5084 2578       5084      1764              0               0    0       0      0 KB
```

* 权限字段多了一个s和p的标记，s表示是和别人共享的内存空间，读写会影响到其他进程，而p表示这是自己私有的内存空间，读写这部分内存不会对其他进程造成影响。

* 输出标示出了[heap]段，并且也说明了后面几个[anon]代表的什么意思（vvar，vdso，vsyscall都是映射到内核的特殊段），mapping字段为空的都是上一行mapping文件里面的bss段（可是gconv-modules.cache后面有两行anonymous mapping，可能跟共享内存有关系，没有深究）。

* Anonymous列标示出了哪些是并且有多少是Anonymous方式映射的物理内存，其大小小于等于RSS

* RSS列表示实际占用的物理内存大小

## top命令输出的SHR内存
最后来看看top命令输出的SHR到底由pmap的哪些输出构成
```bash
dev@dev:~$ pmap -d $$
3108:   bash
Address           Kbytes Mode  Offset           Device    Mapping
0000000000400000     976 r-x-- 0000000000000000 0fc:00000 bash
00000000006f3000       4 r---- 00000000000f3000 0fc:00000 bash
00000000006f4000      36 rw--- 00000000000f4000 0fc:00000 bash
00000000006fd000      24 rw--- 0000000000000000 000:00000   [ anon ]
0000000000c23000    1544 rw--- 0000000000000000 000:00000   [ anon ]
......
00007f53af18e000      16 rw--- 0000000000000000 000:00000   [ anon ]
00007f53af198000       8 rw--- 0000000000000000 000:00000   [ anon ]
00007f53af19a000       4 r---- 0000000000025000 0fc:00000 ld-2.23.so
00007f53af19b000       4 rw--- 0000000000026000 0fc:00000 ld-2.23.so
00007f53af19c000       4 rw--- 0000000000000000 000:00000   [ anon ]
00007ffc5a94b000     132 rw--- 0000000000000000 000:00000   [ stack ]
00007ffc5a9b7000       8 r---- 0000000000000000 000:00000   [ anon ]
00007ffc5a9b9000       8 r-x-- 0000000000000000 000:00000   [ anon ]
ffffffffff600000       4 r-x-- 0000000000000000 000:00000   [ anon ]
mapped: 22464K    writeable/private: 1848K    shared: 28K

dev@dev:~$ top -p $$
  PID USER      PR  NI    VIRT    RES    SHR S %CPU %MEM     TIME+ COMMAND
 3108 dev       20   0   22464   5028   3264 S  0.0  1.0   0:00.02 bash
```
从上面的输出可看出SHR ≈ RES - writeable/private，其中writeable/private主要包含stack和heap以及可执行文件和动态库的data和bss段，而stack+heap=1544+132=1675，这已经占了绝大部分，从而data和bss段之类的基本上可以忽略了，所以一般情况下，SHR ≈ RES - [heap] - [stack]，由于stack一般都比较小，上面的等式可以进一步约等于：SHR ≈ RES - [heap]。

## 总结
top命令能看到一个进程占用的虚拟内存空间、物理内存空间以及和别的进程共享的物理内存空间，这里共享的空间包括通过mmap共享的内存以及共享的可执行文件以及动态库。而mmap命令能看到更详细的信息，比如可执行文件和它所链接的动态库大小，以及物理内存都是被哪些段给占用了。

进程占用的虚拟地址空间大小跟程序的规模有关，除了stack和heap段，其他段的大小基本上都是固定的，并且在程序链接的时候就已经确定了，所以基本上只要关注stack和heap段就可以了，由于stack相对heap来说很小，所以只要没什么stack异常，只需要关注heap。

在实际的工作过程中，其实我们更关心的是RSS用了多少，都被谁用了，简单点说，如果我们没有同时启动多个进程（同一个程序），RSS就是一个很好的实际物理内存使用参考值，但如果是像apache那样同时跑很多个进程，那么RSS减去SHR所占用的空间就是一个很好的实际物理内存占用参考值，当然这都是大概估算值。

要想精确评估一个进程到底占了多少内存，还是很难的，需要对进程的每个段有深入的理解，尤其是SHR部分都有哪些进程在一起共享，不过现在服务器上的内存都是以G为单位的，所以一般情况下大概的估算一下加上合理的测试就能满足我们的需求了。

## 参考
* [Understanding Process memory](https://techtalk.intersec.com/2013/07/memory-part-2-understanding-process-memory/)
* [Understanding memory usage on Linux](http://virtualthreads.blogspot.jp/2006/02/understanding-memory-usage-on-linux.html)