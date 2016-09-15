# Linux Namespace系列（03）：IPC namespace (CLONE_NEWIPC)

IPC namespace用来隔离[System V IPC objects](http://man7.org/linux/man-pages/man7/svipc.7.html)和[POSIX message queues](http://man7.org/linux/man-pages/man7/mq_overview.7.html)。其中System V IPC objects包含Message queues、Semaphore sets和Shared memory segments. 

对于其他几种IPC方式，下面是我的理解，有可能不对，仅供参考，欢迎指正：

* signal没必要隔离，因为它和pid密切相关，当pid隔离后，signal自然就隔离了，能不能跨pid namespace发送signal则由pid namespace决定
* pipe好像也没必要隔离，对匿名pipe来说，只能在父子进程之间通讯，所以隔离的意义不大，而命名管道和文件系统有关，所以只要做好文件系统的隔离，命名管道也就隔离了
* socket和网络接口及协议栈有关，只要网络接口和协议栈隔离了，socket自然也就隔离了

>下面的所有例子都在ubuntu-server-x86_64 16.04下执行通过

##namespace相关tool
从这篇文章开始，不再像介绍UTS namespace那样自己写代码，而是用ubuntu 16.04中现成的两个工具，他们的实现和上一篇文章中介绍UTS namespace时的代码类似，只是多了一些参数处理

* nsenter：加入指定进程的指定类型的namespace，然后执行参数中指定的命令。详情请参考[帮助文档](http://man7.org/linux/man-pages/man1/nsenter.1.html)和[代码](https://github.com/karelzak/util-linux/blob/master/sys-utils/nsenter.c)。
* unshare：离开当前指定类型的namespace，创建且加入新的namespace，然后执行参数中指定的命令。详情请参考[帮助文档](http://man7.org/linux/man-pages/man1/unshare.1.html)和[代码](https://github.com/karelzak/util-linux/blob/master/sys-utils/unshare.c)。

##示例
这里将以消息队列为例，演示一下隔离效果，在本例中将用到两个ipc相关的命令

* ipcmk - 创建shared memory segments, message queues, 和semaphore arrays
* ipcs -  查看shared memory segments, message queues, 和semaphore arrays的相关信息

为了使演示更直观，我们在创建新的ipc namespace的时候，同时也创建新的uts namespace，然后为新的utsnamespace设置新hostname，这样就能通过shell提示符一眼看出这是属于新的namespace的bash，后面的文章中也采取这种方式启动新的bash。

在这个示例中，我们将用到两个shell窗口
```bash
#--------------------------第一个shell窗口----------------------
#记下默认的uts和ipc namespace number
dev@ubuntu:~$ readlink /proc/$$/ns/uts /proc/$$/ns/ipc
uts:[4026531838]
ipc:[4026531839]

#确认hostname
dev@ubuntu:~$ hostname
ubuntu

#查看现有的ipc Message Queues，默认情况下没有message queue
dev@ubuntu:~$ ipcs -q
------ Message Queues --------
key        msqid      owner      perms      used-bytes   messages

#创建一个message queue
dev@ubuntu:~$ ipcmk -Q
Message queue id: 0
dev@ubuntu:~$ ipcs -q
------ Message Queues --------
key        msqid      owner      perms      used-bytes   messages
0x12aa0de5 0          dev        644        0            0


#--------------------------第二个shell窗口----------------------
#重新打开一个shell窗口，确认和上面的shell是在同一个namespace，
#能看到上面创建的message queue
dev@ubuntu:~$ readlink /proc/$$/ns/uts /proc/$$/ns/ipc
uts:[4026531838]
ipc:[4026531839]
dev@ubuntu:~$ ipcs -q
------ Message Queues --------
key        msqid      owner      perms      used-bytes   messages
0x12aa0de5 0          dev        644        0            0

#运行unshare创建新的ipc和uts namespace，并且在新的namespace中启动bash
#这里-i表示启动新的ipc namespace，-u表示启动新的utsnamespace
dev@ubuntu:~$ sudo unshare -iu /bin/bash
root@ubuntu:~#

#确认新的bash已经属于新的ipc和uts namespace了
root@ubuntu:~# readlink /proc/$$/ns/uts /proc/$$/ns/ipc
uts:[4026532455]
ipc:[4026532456]

#设置新的hostname以便和第一个shell里面的bash做区分
root@ubuntu:~# hostname container001
root@ubuntu:~# hostname
container001

#当hostname改变后，bash不会自动修改它的命令行提示符
#所以运行exec bash重新加载bash
root@ubuntu:~# exec bash
root@container001:~#
root@container001:~# hostname
container001

#现在各个bash进程间的关系如下
#bash(24429)是shell窗口打开时的bash
#bash(27668)是运行sudo unshare创建的bash，和bash(24429)不在同一个namespace
root@container001:~# pstree -pl
├──sshd(24351)───sshd(24428)───bash(24429)───sudo(27667)───bash(27668)───pstree(27695)

#查看message queues，看不到原来namespace里面的消息，说明已经被隔离了
root@container001:~# ipcs -q
------ Message Queues --------
key        msqid      owner      perms      used-bytes   messages

#创建一条新的message queue
root@container001:~# ipcmk -Q
Message queue id: 0
root@container001:~# ipcs -q
------ Message Queues --------
key        msqid      owner      perms      used-bytes   messages
0x54b08fc2 0          root       644        0            0

#--------------------------第一个shell窗口----------------------
#回到第一个shell窗口，看看有没有受到新namespace的影响
dev@ubuntu:~$ ipcs -q
------ Message Queues --------
key        msqid      owner      perms      used-bytes   messages
0x12aa0de5 0          dev        644        0            0
#完全无影响，还是原来的信息

#试着加入第二个shell窗口里面bash的uts和ipc namespace
#-t后面跟pid用来指定加入哪个进程所在的namespace
#这里27668是第二个shell中正在运行的bash的pid
#加入成功后将运行/bin/bash
dev@ubuntu:~$ sudo nsenter -t 27668 -u -i /bin/bash

#加入成功，bash的提示符也自动变过来了
root@container001:~# readlink /proc/$$/ns/uts /proc/$$/ns/ipc
uts:[4026532455]
ipc:[4026532456]

#显示的是新namespace里的message queues
root@container001:~# ipcs -q
------ Message Queues --------
key        msqid      owner      perms      used-bytes   messages
0x54b08fc2 0          root       644        0            0

```

##总结
上面介绍了IPC namespace和两个常用的跟namespace相关的工具，从演示过程可以看出，IPC namespace差不多和UTS namespace一样简单，没有太复杂的逻辑，也没有父子namespace关系。不过后续将要介绍的其他namespace就要比这个复杂多了。