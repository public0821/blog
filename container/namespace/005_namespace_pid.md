# Linux Namespace系列（05）：pid namespace (CLONE_NEWPID)

PID namespaces用来隔离进程的ID空间，使得不同pid namespace里的进程ID可以重复且相互之间不影响。 

PID namespace可以嵌套，也就是说有父子关系，在当前namespace里面创建的所有新的namespace都是当前namespace的子namespace。父namespace里面可以看到所有子孙后代namespace里的进程信息，而子namespace里看不到祖先或者兄弟namespace里的进程信息。

目前PID namespace最多可以嵌套32层，由内核中的宏MAX_PID_NS_LEVEL来定义

Linux下的每个进程都有一个对应的/proc/PID目录，该目录包含了大量的有关当前进程的信息。 对一个PID namespace而言，/proc目录只包含当前namespace和它所有子孙后代namespace里的进程的信息。

在Linux系统中，进程ID从1开始往后不断增加，并且不能重复（当然进程退出后，ID会被回收再利用），进程ID为1的进程是内核启动的第一个应用层进程，一般是init进程（现在采用systemd的系统第一个进程是systemd），具有特殊意义，当系统中一个进程的父进程退出时，内核会指定init进程成为这个进程的新父进程，而当init进程退出时，系统也将退出。

除了在init进程里指定了handler的信号外，内核会帮init进程屏蔽掉其他任何信号，这样可以防止其他进程不小心kill掉init进程导致系统挂掉。不过有了PID namespace后，可以通过在父namespace中发送SIGKILL或者SIGSTOP信号来终止子namespace中的ID为1的进程。

由于ID为1的进程的特殊性，所以每个PID namespace的第一个进程的ID都是1。当这个进程运行停止后，内核将会给这个namespace里的所有其他进程发送SIGKILL信号，致使其他所有进程都停止，于是namespace被销毁掉。

>本篇所有例子都在ubuntu-server-x86_64 16.04下执行通过

## 简单示例

```bash
#查看当前pid namespace的ID
dev@ubuntu:~$ readlink /proc/self/ns/pid
pid:[4026531836]

#启动新的pid namespace
#这里同时也启动了新的uts和mount namespace
#新的uts是为了设置一个新的hostname，便于和老的namespace区分
#新的mount namespace是为了方便我们修改新namespace里面的mount信息，
#因为这样不会对老namespace造成影响
#这里--fork是为了让unshare进程fork一个新的进程出来，然后再用bash替换掉新的进程
#这是pid namespace本身的限制，进程所属的pid namespace在它创建的时候就确定了，不能更改，
#所以调用unshare和nsenter后，原来的进程还是属于老的namespace，
#而新fork出来的进程才属于新的namespace
dev@ubuntu:~$ sudo unshare --uts --pid --mount --fork /bin/bash
root@ubuntu:~# hostname container001
root@ubuntu:~# exec bash
root@container001:~#

#查看进程间关系，当前bash(31646)确实是unshare的子进程
root@container001:~# pstree -pl
├─sshd(955)─┬─sshd(17810)───sshd(17891)───bash(17892)───sudo(31644)──
─unshare(31645)───bash(31646)───pstree(31677)
#他们属于不同的pid namespace
root@container001:~# readlink /proc/31645/ns/pid
pid:[4026531836]
root@container001:~# readlink /proc/31646/ns/pid
pid:[4026532469]

#但为什么通过这种方式查看到的namespace还是老的呢？
root@container001:~# readlink /proc/$$/ns/pid
pid:[4026531836]

#由于我们实际上已经是在新的namespace里了，并且当前bash是当前namespace的第一个进程
#所以在新的namespace里看到的他的进程ID是1
root@container001:~# echo $$
1
#但由于我们新的namespace的挂载信息是从老的namespace拷贝过来的，
#所以这里看到的还是老namespace里面的进程号为1的信息
root@container001:~# readlink /proc/1/ns/pid
pid:[4026531836]
#ps命令依赖/proc目录，所以ps的输出还是老namespace的视图
root@container001:~# ps ef
UID        PID  PPID  C STIME TTY          TIME CMD
root         1     0  0 7月07 ?       00:00:06 /sbin/init
root         2     0  0 7月07 ?       00:00:00 [kthreadd]
 ...
root     31644 17892  0 7月14 pts/0   00:00:00 sudo unshare --uts --pid --mount --fork /bin/bash
root     31645 31644  0 7月14 pts/0   00:00:00 unshare --uts --pid --mount --fork /bin/bash

#所以我们需要重新挂载我们的/proc目录
root@container001:~# mount -t proc proc /proc

#重新挂载后，能看到我们新的pid namespace ID了
root@container001:~# readlink /proc/$$/ns/pid
pid:[4026532469]
#ps的输出也正常了
root@container001:~# ps -ef
UID        PID  PPID  C STIME TTY          TIME CMD
root         1     0  0 7月14 pts/0   00:00:00 bash
root        44     1  0 00:06 pts/0    00:00:00 ps -ef
```

## PID namespace嵌套

* 调用unshare或者setns函数后，当前进程的namespace不会发生变化，不会加入到新的namespace，而它的子进程会加入到新的namespace。也就是说进程属于哪个namespace是在进程创建的时候决定的，并且以后再也无法更改。

* 在一个PID namespace里的进程，它的父进程可能不在当前namespace中，而是在外面的namespace里面（这里外面的namespace指当前namespace的祖先namespace），这类进程的ppid都是0。比如新namespace里面的第一个进程，他的父进程就在外面的namespace里。通过setns的方式加入到新namespace中的进程的父进程也在外面的namespace中。
 
* 可以在祖先namespace中看到子namespace的所有进程信息，且可以发信号给子namespace的进程，但进程在不同namespace中的PID是不一样的。

### 嵌套示例
```bash
#--------------------------第一个shell窗口----------------------
#记下最外层的namespace ID
dev@ubuntu:~$ readlink /proc/$$/ns/pid
pid:[4026531836]

#创建新的pid namespace， 这里--mount-proc参数是让unshare自动重新mount /proc目录
dev@ubuntu:~$ sudo unshare --uts --pid --mount --fork --mount-proc /bin/bash
root@ubuntu:~# hostname container001
root@ubuntu:~# exec bash
root@container001:~# readlink /proc/$$/ns/pid
pid:[4026532469]

#再创建新的pid namespace
root@container001:~# unshare --uts --pid --mount --fork --mount-proc /bin/bash
root@container001:~# hostname container002
root@container001:~# exec bash
root@container002:~# readlink /proc/$$/ns/pid
pid:[4026532472]

#再创建新的pid namespace
root@container002:~# unshare --uts --pid --mount --fork --mount-proc /bin/bash
root@container002:~# hostname container003
root@container002:~# exec bash
root@container003:~# readlink /proc/$$/ns/pid
pid:[4026532475]

#目前namespace container003里面就一个bash进程
root@container003:~# pstree -p
bash(1)───pstree(22)
#这样我们就有了三层pid namespace，
#他们的父子关系为container001->container002->container003

#--------------------------第二个shell窗口----------------------
#在最外层的namespace中查看上面新创建的三个namespace中的bash进程
#从这里可以看出，这里显示的bash进程的PID和上面container003里看到的bash(1)不一样
dev@ubuntu:~$ pstree -pl|grep bash|grep unshare
|-sshd(955)-+-sshd(17810)---sshd(17891)---bash(17892)---sudo(31814)--
-unshare(31815)---bash(31816)---unshare(31842)---bash(31843)--
-unshare(31864)---bash(31865)
#各个unshare进程的子bash进程分别属于上面的三个pid namespace
dev@ubuntu:~$ sudo readlink /proc/31816/ns/pid
pid:[4026532469]
dev@ubuntu:~$ sudo readlink /proc/31843/ns/pid
pid:[4026532472]
dev@ubuntu:~$ sudo readlink /proc/31865/ns/pid
pid:[4026532475]

#PID在各个namespace里的映射关系可以通过/proc/[pid]/status查看到
#这里31865是在最外面namespace中看到的pid
#45，23，1分别是在container001，container002和container003中的pid
dev@ubuntu:~$ grep pid /proc/31865/status
NSpid:  31865   45     23      1

#创建一个新的bash并加入container002
dev@ubuntu:~$ sudo nsenter --uts --mount --pid -t 31843 /bin/bash
root@container002:/#

#这里bash（23）就是container003里面的pid 1对应的bash
root@container002:/# pstree -p
bash(1)───unshare(22)───bash(23)
#unshare(22)属于container002
root@container002:/# readlink /proc/22/ns/pid
pid:[4026532472]
#bash（23）属于container003
root@container002:/# readlink /proc/23/ns/pid
pid:[4026532475]

#为什么上面pstree的结果里面没看到nsenter加进来的bash呢？
#通过ps命令我们发现，我们新加进来的那个/bin/bash的ppid是0，难怪pstree里面显示不出来
#从这里可以看出，跟最外层namespace不一样的地方就是，这里可以有多个进程的ppid为0
#从这里的TTY也可以看出哪些命令是在哪些窗口执行的，
#pts/0对应第一个shell窗口，pts/1对应第二个shell窗口
root@container002:/# ps -ef
UID        PID  PPID  C STIME TTY          TIME CMD
root         1     0  0 04:39 pts/0    00:00:00 bash
root        22     1  0 04:39 pts/0    00:00:00 unshare --uts --pid --mount --fork --mount-proc /bin/bash
root        23    22  0 04:39 pts/0    00:00:00 bash
root        46     0  0 04:52 pts/1    00:00:00 /bin/bash
root        59    46  0 04:53 pts/1    00:00:00 ps -ef

#--------------------------第三个shell窗口----------------------
#创建一个新的bash并加入container001
dev@ubuntu:~$ sudo nsenter --uts --mount --pid -t 31816 /bin/bash
root@container001:/#

#通过pstree和ps -ef我们可看到所有三个namespace中的进程及他们的关系
#bash(1)───unshare(22)属于container001
#bash(23)───unshare(44)属于container002
#bash(45)属于container003，而68和84两个进程分别是上面两次通过nsenter加进来的bash
#同上面ps的结果比较我们可以看出，同样的进程在不同的namespace里面拥有不同的PID
root@container001:/# pstree -pl
bash(1)───unshare(22)───bash(23)───unshare(44)───bash(45)
root@container001:/# ps -ef
UID        PID  PPID  C STIME TTY          TIME CMD
root         1     0  0 04:37 pts/0    00:00:00 bash
root        22     1  0 04:39 pts/0    00:00:00 unshare --uts --pid --mount --fork --mount-proc /bin/bash
root        23    22  0 04:39 pts/0    00:00:00 bash
root        44    23  0 04:39 pts/0    00:00:00 unshare --uts --pid --mount --fork --mount-proc /bin/bash
root        45    44  0 04:39 pts/0    00:00:00 bash
root        68     0  0 04:52 pts/1    00:00:00 /bin/bash
root        84     0  0 05:00 pts/2    00:00:00 /bin/bash
root        95    84  0 05:00 pts/2    00:00:00 ps -ef

#发送信号给contain002中的bash
root@container001:/# kill 68

#--------------------------第二个shell窗口----------------------
#回到第二个窗口，发现bash已经被kill掉了，说明父namespace是可以发信号给子namespace中的进程的
root@container002:/# exit
dev@ubuntu:~$
```

## “init”示例
当一个进程的父进程被kill掉后，该进程将会被当前namespace中pid为1的进程接管，而不是被最外层的系统级别的init进程接管。

当pid为1的进程停止运行后，内核将会给这个namespace及其子孙namespace里的所有其他进程发送SIGKILL信号，致使其他所有进程都停止，于是当前namespace及其子孙后代的namespace都被销毁掉。

```bash 
#还是继续以上面三个namespace为例
#--------------------------第一个shell窗口----------------------
#在003里面启动两个新的bash，使他们的继承关系如下
root@container003:~# bash
root@container003:~# bash
root@container003:~# pstree
bash───bash───bash───pstree

#利用unshare、nohup和sleep的组合，模拟出我们想要的父子进程
#unshare --fork会使unshare创建一个子进程
#nohup sleep sleep 3600&会让这个子进程在后台运行并且sleep一小时
root@container003:~# unshare --fork nohup sleep 3600&
[1] 77

#于是我们得到了我们想要的进程间关系结构
root@container003:~# pstree -p
bash(1)───bash(26)───bash(36)─┬─pstree(80)
                              └─unshare(77)───sleep(78)

#如我们所期望的，kill掉unshare(77)后， sleep就被当前pid namespace的bash(1)接管了
root@container003:~# kill 77
root@container003:~# pstree -p
bash(1)─┬─bash(26)───bash(36)───pstree(82)
        └─sleep(78)

#重新回到刚才的状态，后面将尝试在第三个窗口中kill掉这里的unshare进程
root@container003:~# kill 78
root@container003:~# unshare --fork nohup sleep 3600&
root@container003:~# pstree -p
bash(1)───bash(26)───bash(36)─┬─pstree(85)
                              └─unshare(83)───sleep(84)

#--------------------------第三个shell窗口----------------------
#来到第三个窗口
root@container001:/# pstree -p
bash(1)───unshare(22)───bash(23)───unshare(44)───bash(45)───bash(113)─
──bash(123)───unshare(170)───sleep(171)

#kill掉sleep(171)的父近程unshare(170)，
root@container001:/# kill 170
#结果显示sleep（171）被bash(45)接管了，而不是bash(1)，
#进一步说明container003里的进程只会被container003里的pid 1进程接管，
#而不会被外面container001的pid 1进程接管
root@container001:/# pstree -p
bash(1)───unshare(22)───bash(23)───unshare(44)──
─bash(45)─┬─bash(113)───bash(123)
          └─sleep(171)

#kill掉container002中pid 1的bash进程，在container001中，对应的是bash(23)
root@container001:/# kill 23
#根本没反应，说明bash不接收TERM信号（kill默认发送SIGTERM信号）
root@container001:/# pstree -p
bash(1)───unshare(22)───bash(23)───unshare(44)──
─bash(45)─┬─bash(113)───bash(123)
          └─sleep(171)

#试试SIGSTOP，貌似也不行      
root@container001:/# kill -SIGSTOP 23
root@container001:/# pstree -p
bash(1)───unshare(22)───bash(23)───unshare(44)──
─bash(45)─┬─bash(113)───bash(123)
          └─sleep(171)

#最后试试杀手锏SIGKILL，马到成功
root@container001:/# kill -SIGKILL 23
root@container001:/# pstree -p
bash(1)

#--------------------------第一个shell窗口----------------------
#container003和container002的bash退出了，
#第一个shell窗口直接退到了container001的bash
root@container003:~# Killed
root@container001:~#

#--------------------------第二个shell窗口----------------------
#通过nsenter方式加入到container002的bash也被kill掉了
root@container002:/# Killed
dev@ubuntu:~$

#从结果可以看出，container002的“init”进程被杀死后，
#内核将会发送SIGKILL给container002里的所有进程，
#这样导致container002及它所有子孙namespace里的进程都杀死，
#同时container002和container003也被销毁
```

[man-pages](http://man7.org/linux/man-pages/man7/pid_namespaces.7.html)里面说SIGSTOP也可以kill掉子namespace里的“init”进程，但我在上面试了下，没效果，具体原因未知。

## 其他

* 通常情况下，如果PID namespace中的进程都退出了，这个namespace将会被销毁，但就如在前面[“Namespace概述”](001_namespace_introduction.md)里介绍的，有两种情况会导致就算进程都退出了，这个namespace还会存在。但对于PID namespace来说，就算namespace还在，由于里面没有“init”进程，Kernel不允许其它进程加入到这个namespace，所以这个存在的namespace没有意义

* 当一个PID通过UNIX domain socket在不同的PID namespace中传输时（请参考[unix(7)](http://man7.org/linux/man-pages/man7/unix.7.html)里面的SCM_CREDENTIALS），PID将会自动转换成目的namespace中的PID.

## 参考

* [Namespaces in operation, part 3: PID namespaces](https://lwn.net/Articles/531419/)
* [Namespaces in operation, part 4: more on PID namespaces](https://lwn.net/Articles/532748/)
* [overview of Linux PID namespaces](http://man7.org/linux/man-pages/man7/pid_namespaces.7.html)