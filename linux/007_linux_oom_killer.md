#Linux OOM killer

作为Linux下的程序员，有时不得不面对一个问题，那就是系统内存被用光了，这时当进程再向内核申请内存时，内核会怎么办呢？程序里面调用的malloc函数会返回null吗？

为了处理内存不足时的问题，Linux内核发明了一种机制，叫OOM(Out Of Memory) killer，通过配置它可以控制内存不足时内核的行为。

##OOM killer
当物理内存和交换空间都被用完时，如果还有进程来申请内存，内核将触发OOM killer，其行为如下：

1.检查文件/proc/sys/vm/panic_on_oom，如果里面的值为2，那么系统一定会触发panic
2.如果/proc/sys/vm/panic_on_oom的值为1，那么系统有可能触发panic（见后面的介绍）
3.如果/proc/sys/vm/panic_on_oom的值为0，或者上一步没有触发panic，那么内核继续检查文件/proc/sys/vm/oom_kill_allocating_task
3.如果/proc/sys/vm/oom_kill_allocating_task为1，那么内核将kill掉当前申请内存的进程
4.如果/proc/sys/vm/oom_kill_allocating_task为0，内核将检查每个进程的分数，分数最高的进程将被kill掉（见后面介绍）

进程被kill掉之后，如果/proc/sys/vm/oom_dump_tasks为1，且系统的rlimit中设置了core文件大小，将会由/proc/sys/kernel/core_pattern里面指定的程序生成core dump文件，这个文件里将包含
pid, uid, tgid, vm size, rss, nr_ptes, nr_pmds, swapents, oom_score_adj
score, name等内容，拿到这个core文件之后，可以做一些分析，看为什么这个进程被选中kill掉。

这里可以看看ubuntu默认的配置：
```bash
#OOM后不panic
dev@ubuntu:~$ cat /proc/sys/vm/panic_on_oom
0

#OOM后kill掉分数最高的进程
dev@ubuntu:~$ cat /proc/sys/vm/oom_kill_allocating_task
0

#进程由于OOM被kill掉后将生成core dump文件
dev@ubuntu:~$ cat /proc/sys/vm/oom_dump_tasks
1

#默认max core file size是0， 所以系统不会生成core文件
dev@ubuntu:~$ prlimit|grep CORE
CORE max core file size 0 unlimited blocks

#core dump文件的生成交给了apport，相关的设置可以参考apport的资料
dev@ubuntu:~$ cat /proc/sys/kernel/core_pattern
|/usr/share/apport/apport %p %s %c %P
```
参考：[apport](https://wiki.ubuntu.com/Apport)

###panic_on_oom
正如上面所介绍的那样，该文件的值可以取0/1/2，0是不触发panlic，2是一定触发panlic，如果为1的话就要看[mempolicy](https://www.kernel.org/doc/Documentation/vm/numa_memory_policy.txt)和[cpusets](https://www.kernel.org/doc/Documentation/cgroup-v1/cpusets.txt)，这篇不介绍这方面的内容。

panic后内核的默认行
为是死在那里，目的是给开发人员一个连上去debug的机会。但对于大多数应用层开发人员来说没啥用，倒是希望它赶紧重启。为了让内核panic后重启，可以修改文件/proc/sys/kernel/panic，里面表示的是panic多少秒后系统将重启，这个文件的默认值是0，表示永远不重启。
```bash
#设置panic后3秒重启系统
dev@ubuntu:~$ sudo sh -c "echo 3 > /proc/sys/kernel/panic"
```

###调整分数
当oom_kill_allocating_task的值为0时（系统默认配置），系统会kill掉系统中分数最高的那个进程，这里的分数是怎么来的呢？该值由内核维护，并存储在每个进程的/proc/<pid>/oom_score文件中。

每个进程的分数受多方面的影响，比如进程运行的时间，时间越长表明这个程序越重要，所以分数越低；进程从启动后分配的内存越多，表示越占内存，分数会越高；这里只是列举了一两个影响分数的因素，实际情况要复杂的多，需要看内核代码，这里有篇文章可以参考：[Taming the OOM killer](https://lwn.net/Articles/317814/)

由于分数计算复杂，比较难控制，于是内核提供了另一个文件用来调控分数，那就是文件/proc/<pid>/oom_adj，这个文件的默认值是0，但它可以配置为-17到15中间的任何一个值，内核在计算了进程的分数后，会和这个文件的值进行一个计算，得到的结果会作为进程的最终分数写入/proc/<pid>/oom_score。计算方式大概如下：

* 如果/proc/<pid>/oom_adj的值为正数，那么分数将会被乘以2的n次方，这里n是文件里面的值
* 如果/proc/<pid>/oom_adj的值为负数，那么分数将会被除以2的n次方，这里n是文件里面的值

由于进程的分数在内核中是一个16位的整数，所以-17就意味着最终进程的分数永远是0，也即永远不会被kill掉。

当然这种控制方式也不是非常精确，但至少比没有强多了。

###修改配置
上面的这些文件都可以通过下面三种方式来修改，这里以panic_on_oom为例做个示范：

* 直接写文件（重启后失效）
    ```bash
    dev@ubuntu:~$ sudo sh -c "echo 2> /proc/sys/vm/panic_on_oom"
    ```

* 通过控制命令（重启后失效）
    ```bash
    dev@dev:~$ sudo sysctl vm.panic_on_oom=2
    ```

* 修改配置文件（重启后继续生效）
    ```bash
    #通过编辑器将vm.panic_on_oom=2添加到文件sysctl.conf中（如果已经存在，修改该配置项即可）
    dev@dev:~$ sudo vim /etc/sysctl.conf

    #重新加载sysctl.conf，使修改立即生效
    dev@dev:~$ sudo sysctl -p
    ```

###日志
一旦OOM killer被触发，内核将会生成相应的日志，一般可以在/var/log/messages里面看到，如果配置了syslog，日志可能在/var/log/syslog里面，这里是ubuntu里的日志样例
```bash
dev@dev:~$ grep oom /var/log/syslog
Jan 23 21:30:29 dev kernel: [  490.006836] eat_memory invoked oom-killer: gfp_mask=0x24280ca, order=0, oom_score_adj=0
Jan 23 21:30:29 dev kernel: [  490.006871]  [<ffffffff81191442>] oom_kill_process+0x202/0x3c0
```

##cgroup的OOM killer
除了系统的OOM killer之外，如果配置了memory cgroup，那么进程还将受到自己所属memory cgroup的限制，如果超过了cgroup的限制，将会触发cgroup的OOM killer，cgroup的OOM killer和系统的OOM killer行为略有不同，详情请参考[Linux Cgroup系列（04）：限制cgroup的内存使用](https://segmentfault.com/a/1190000008125359)。

##malloc
malloc是libc的函数，C/C++程序员对这个函数应该都很熟悉，它里面实际上调用的是内核的[sbrk](http://man7.org/linux/man-pages/man2/brk.2.html)和[mmap](http://man7.org/linux/man-pages/man2/mmap.2.html)，为了避免频繁的调用内核函数和优化性能，它里面在内核函数的基础上实现了一套自己的内存管理功能。

既然内存不够时有OOM killer帮我们kill进程，那么这时调用的malloc还会返回NULL给应用进程吗？答案是不会，因为这时只有两种情况：

1. 当前申请内存的进程被kill掉：都被kill掉了，返回什么都没有意义了
2. 其它进程被kill掉：释放出了空闲的内存，于是内核就能给当前进程分配内存了

那什么时候我们调用malloc的时候会返回NULL呢，从malloc函数的[帮助文件](http://man7.org/linux/man-pages/man3/malloc.3.html)可以看出，下面两种情况会返回NULL：

* 使用的虚拟地址空间超过了RLIMIT_AS的限制
* 使用的数据空间超过了RLIMIT_DATA的限制，这里的数据空间包括程序的数据段，BSS段以及heap

关于虚拟地址空间和heap之类的介绍请参考[Linux进程的内存使用情况](https://segmentfault.com/a/1190000008125059)，这两个参数的默认值为unlimited，所以只要不修改它们的默认配置，限制就不会被触发。有一种极端情况需要注意，那就是代码写的有问题，超过了系统的虚拟地址空间范围，比如32位系统的虚拟地址空间范围只有4G，这种情况下不确定系统会以一种什么样的方式返回错误。

##rlimit
上面提到的RLIMIT_AS和RLIMIT_DATA都可以通过函数[getrlimit和setrlimit](http://man7.org/linux/man-pages/man2/getrlimit.2.html)来设置和读取，同时linux还提供了一个[prlimit](http://man7.org/linux/man-pages/man1/prlimit.1.html)程序来设置和读取rlimit的配置。

prlimit是用来替代
[ulimit](http://man7.org/linux/man-pages/man3/ulimit.3.html)的一个程序，除了能设置上面的那两个参数之外，还有其它的一些参数，比如core文件的大小。关于prlimit的用法请参考它的[帮助文件](http://man7.org/linux/man-pages/man1/prlimit.1.html)。

```bash
#默认情况下，RLIMIT_AS和RLIMIT_DATA的值都是unlimited
dev@dev:~$ prlimit |egrep "DATA|AS"
AS         address space limit                unlimited unlimited bytes
DATA       max data size                      unlimited unlimited bytes
```

##测试代码
C语言的程序会受到libc的影响，可能在触发OOM killer之前就触发了segmentfault错误，如果要用C语言程序来测试触发OOM killer，一定要注意malloc的行为受MMAP_THRESHOLD影响，一次申请分配太多内存的话，malloc会调用mmap映射内存，从而不一定触发OOM killer，具体细节目前还不太清楚。这里是一个触发oom killer的例子，供参考：
```bash
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>

#define M (1024 * 1024)
#define K 1024

int main(int argc, char *argv[])
{
    char *p;
    int size =0;
    while(1) {
        p = (char *)malloc(K);
        if  (p == NULL){
            printf("memory allocate failed!\n");
            return -1;
        }
        memset(p, 0, K);
        size += K;
        if (size%(100*M) == 0){
            printf("%d00M memory allocated\n", size/(100*M));
            sleep(1);
        }
    }

    return 0;
}
```

##总结
对一个进程来说，内存的使用受多种因素的限制，可能在系统内存不足之前就达到了rlimit和memory cgroup的限制，同时它还可能受不同编程语言所使用的相关内存管理库的影响，就算系统处于内存不足状态，申请新内存也不一定会触发OOM killer，需要具体问题具体分析。

##参考
[sysctl/vm.txt](https://www.kernel.org/doc/Documentation/sysctl/vm.txt)
[How to Configure the Linux Out-of-Memory Killer](http://www.oracle.com/technetwork/articles/servers-storage-dev/oom-killer-1911807.html)
[When Linux Runs Out of Memory](http://www.linuxdevcenter.com/pub/a/linux/2006/11/30/linux-out-of-memory.html?page=2)