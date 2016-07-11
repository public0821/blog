# UTS namespaces (CLONE_NEWUTS)
UTS namespaces提供了对系统hostname以及NIS domain name的隔离. 这两个资源可以通过sethostname(2) and setdomainname(2)函数来设置，以及通过uname(2), gethostname(2), and getdomainname(2)函数来获取.（这里括号中的2表示这个函数是system call，具体其他数字的含义请参看man的帮助文件）

术语UTS来自于调用函数uname()时用到的结构体: struct utsname. 而这个结构体的名字源自于"UNIX Time-sharing System".

由于UTS namespace最简单，所以放在最前面介绍，在这篇文章中我们将会熟悉UTS  namespace以及和namespace相关的三个系统调用的使用。

注意：
NIS domain name和DNS没有关系，关于他的介绍可以看[这里](https://www.freebsd.org/doc/handbook/network-nis.html)，由于本人对它不了解，所以在本文中不做介绍。

###创建新的UTS namespace
多说无益，直接上代码，我尽量将注释写的足够详细，请仔细看代码和输出结果

**注意:**

* 为了代码简单起见，只在clone函数那做了错误处理，关于clone函数的详细介绍请参考[man-pages](http://man7.org/linux/man-pages/man2/clone.2.html)
* 为了描述方便，某些地方会用hostname来区分UTS namespace，如hostname为aaa的namespace，我们会描述成namespace aaa。

```c
#define _GNU_SOURCE
#include <sched.h>
#include <sys/wait.h>
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <string.h>

#define NOT_OK_EXIT(code, msg); {if(code == -1){perror(msg); exit(-1);} }

//子进程从这里开始执行
static int child_func(void *arg)
{
    //设置主机名
    sethostname(arg, strlen(arg));

    //启动一个新的bash子进程，当前子进程会阻塞在这里直到bash退出
    //由于默认情况下子进程会继承父进程的namespace，
    //所以新启动的bash进程会和当前子进程属于同一个namespace
    system("/bin/bash");

    return 0;    //这行执行完之后，子进程结束
}

static char child_stack[1024*1024]; //设置子进程的栈空间为1M

int main(int argc, char *argv[])
{
    pid_t child_pid;

    if (argc < 2) {
        printf("Usage: %s <child-hostname>\n", argv[0]);
        return -1;
    }

    //创建并启动子进程，调用该函数后，父进程将继续往后执行，也就是执行后面的waitpid函数
    child_pid = clone(
                  //子进程将执行child_func这个函数，当这个函数退出时，子进程结束
                  child_func,  
                  //栈是从高位向低位增长，所以这里要指向高位地址
                  child_stack + sizeof(child_stack),   
                  //创建新的UTS namespace，这里SIGCHLD是子进程退出后返回给父进程的信号
                  CLONE_NEWUTS | SIGCHLD,  
                  argv[1]);  //传给child_func的参数
    NOT_OK_EXIT(child_pid, "clone");

    waitpid(child_pid, NULL, 0); //等待子进程结束

    return 0;    //这行执行完之后，父进程结束
}
```

在上面的代码中：

* 父进程创建新的子进程，并且设置CLONE_NEWUTS，这样就会创建新的UTS namespace并且让子进程属于这个新的namespace，然后父进程一直等待子进程退出
* 子进程在设置好新的hostname后启动了一个新的bash，等待新的bash退出
* 当启动的bash退出后，子进程退出，接着父进程也退出

下面看看输出效果
```bash
#------------------------第一个shell窗口------------------------
#将上面的代码保持为namespace_uts_demo.c， 
#然后用gcc将它编译成可执行文件namespace_uts_demo
dev@ubuntu:~/code$ gcc namespace_uts_demo.c -o namespace_uts_demo   

#创建新的UTS namespace需要root权限，这里先切换到root账户，便于后面的操作 
dev@ubuntu:~/code$ su     
Password:

#启动程序，传入参数container001
#这样新创建的UTS namespace的hostname将被设置成container001
root@ubuntu:~/code# ./namespace_uts_demo container001

#新的bash被启动，从shell的提示符可以看出，hostname已经被改成了container001
root@container001:~/code# 
#用hostname命令再确认一下
root@container001:~/code# hostname
container001

#pstree是用来查看系统中进程之间父子关系的工具
#下面的输出过滤掉了跟namespace_uts_demo无关的内容
#本次操作是通过ssh客户端远程连接到Linux主机进行的，
#所以bash(17892)的父进程是一系列的sshd进程，
#默认我们用dev账号登陆，然后使用su命令，
#于是有了后面的su进程以及由su生成的具有root权限的bash(23101)。
#我们在bash(23101)里面执行了我们的程序，于是有了namespace_uts_d(23111)进程，
#我们的程序自己clone了一个新的进程namespace_uts_d(23112)，
#并且这个进程属于一个新的UTS namespace， 
#在我们的子进程中，我们调用了system("/bin/bash")，于是有了子进程sh(23113)，
#它负责创建了我们想要的bash(23114)进程。
#由于我们的pstree命令是在bash(23114)里面运行的，
#所以这里pstree(23280)是bash(23114)的子进程
root@container001:~/code# pstree -pl
systemd(1)───sshd(955)─┬─sshd(17810)───sshd(17891)───bash(17892)───su(23100)──
─bash(23101)───namespace_uts_d(23111)───namespace_uts_d(23112)───sh(23113)──
─bash(23114)───pstree(23280)

#验证一下我们运行的bash进程是不是bash(23114)
#下面这个命令可以输出当前bash的PID
root@container001:~/code# echo $$
23114

#验证一下我们的父进程和子进程是否不在同一个UTS namespace
root@container001:~/code# readlink /proc/23111/ns/uts
uts:[4026531838]
root@container001:~/code# readlink /proc/23112/ns/uts
uts:[4026532445]
#果然不属于同一个UTS namespace

#默认情况下，子进程应该继承父进程的namespace
#systemd(1)是我们程序父进程namespace_uts_d(23111)的祖先进程，
#他们属于同一个namespace
root@container001:~/code# readlink /proc/1/ns/uts
uts:[4026531838]

#sh(23113)和bash(23114)是我们程序子进程namespace_uts_d(23112)的子进程，
#他们和namespace_uts_d(23112)属于同一个namespace
root@container001:~/code# readlink /proc/23113/ns/uts
uts:[4026532445]
root@container001:~/code# readlink /proc/$$/ns/uts
uts:[4026532445]

#------------------------第二个shell窗口------------------------
#重新打开一个新的shell窗口，这个shell和systemd属于同一个namespace
dev@ubuntu:~/code$ sudo readlink /proc/$$/ns/uts
uts:[4026531838]
#老的namespace中的hostname还是原来的，不受新的namespace影响
dev@ubuntu:~/code$ hostname     
ubuntu
#有兴趣的同学可以在两个shell窗口里面分别用命令hostname设置hostname试试，
#会发现他们两个之间相互不受影响，这里就不演示了


#------------------------第一个shell窗口------------------------
#继续回到原来的shell，试试在container001里面再运行一下那个程序会怎样
root@container001:~/code# ./namespace_uts_demo container002

#创建了一个新的UTS namespace，hostname被改成了container002
root@container002:~/code#
root@container002:~/code# hostname
container002

#新的UTS namespace
root@container002:~/code# readlink /proc/$$/ns/uts
uts:[4026532455]

#进程间的关系和上面的差不多，在后面又生成了一串的namespace_uts_demo进程
root@container002:~/code# pstree -pl
systemd(1)───sshd(955)─┬─sshd(17810)───sshd(17891)───bash(17892)───su(23100)──
─bash(23101)───namespace_uts_d(23111)───namespace_uts_d(23112)───sh(23113)──
─bash(23114)───namespace_uts_d(23297)───namespace_uts_d(23298)───sh(23299)──
─bash(23300)───pstree(23310)

#退出bash(23300)后，
#sh(23299)、namespace_uts_d(23298)、namespace_uts_d(23297)相续退出，
#于是又回到了进程bash(23114)中，hostname于是也回到了container001
#注意： 在bash(23300)退出的过程中，并没有进程的namespace发生变化，
#只是所有属于namespace container002的进程都执行完退出了
root@container002:~/code# exit
exit
root@container001:~/code#
root@container001:~/code# hostname
container001
```

###将当前进程加入指定的namespace
还是直接上代码，有了前面的铺垫，这里的代码就非常简单了，请仔细看代码和输出结果
```c
#define _GNU_SOURCE
#include <fcntl.h>
#include <sched.h>
#include <stdlib.h>
#include <stdio.h>

#define NOT_OK_EXIT(code, msg); {if(code == -1){perror(msg); exit(-1);} }

int main(int argc, char *argv[])
{
    int fd, ret;

    if (argc < 2) {
        printf("%s /proc/PID/ns/FILE\n", argv[0]);
        return -1;
    }

    //根据传入的参数获取namespace对应文件的描述符
    fd = open(argv[1], O_RDONLY);
    NOT_OK_EXIT(fd, "open");

    //使当前进程加入参数中指定的namespace
    //这里第二个参数为0，表示由系统自己检测fd对应的是哪种类型的namespace
    ret = setns(fd, 0);
    NOT_OK_EXIT(ret, "open");

    //启动bash子进程，由于子进程会默认继承父进程的namespace，
    //所以新启动的bash也将会加入上面指定的namespace
    system("/bin/bash");

    return -1;
}
```
在上面的代码中，程序通过setns调用让自己加入到参数指定的namespace中，然后创建bash子进程

再来看结果
```bash
#--------------------------第一个shell窗口----------------------
#还是以上面创建的container001为例
#先确认一下hostname是否正确，
root@container001:~/code# hostname
container001

#获取bash的PID
root@container001:~/code# echo $$
23114

#得到bash所属的UTS namespace
root@container001:~/code# readlink /proc/23114/ns/uts
uts:[4026532445]



#--------------------------第二个shell窗口----------------------
#重新打开一个shell窗口，将上面的代码保存为文件namespace_join.c并编译
dev@ubuntu:~/code$ gcc namespace_join.c -o namespace_join

#运行程序前，确认下当前bash是属于哪个namespace
dev@ubuntu:~/code$ hostname
ubuntu
dev@ubuntu:~/code$ readlink /proc/$$/ns/uts
uts:[4026531838]

#执行程序，使其加入第一个shell窗口中的bash所在的namespace
dev@ubuntu:~/code$ sudo ./namespace_join /proc/23114/ns/uts

#加入成功，bash的hostname和UTS namespace的number和第一个shell窗口的都一样
#这里bash的提示符是‘#’，表示bash有root权限，
#这是因为我们是用sudo来运行的程序，于是我们程序创建的子进程也有root权限
root@container001:~/code# hostname
container001
root@container001:~/code# echo $$
24246
root@container001:~/code# readlink /proc/24246/ns/uts
uts:[4026532445]

#我们自己的程序也是一样，成功的加入了指定的namespace
root@container001:~/code# readlink /proc/`pidof namespace_join`/ns/uts
uts:[4026532445]
```

###退出当前namespace并加入新创建的namespace
继续看代码
```c
#define _GNU_SOURCE
#include <sched.h>
#include <unistd.h>
#include <stdlib.h>
#include <stdio.h>

#define NOT_OK_EXIT(code, msg); {if(code == -1){perror(msg); exit(-1);} }

static void usage(const char *pname)
{
    char usage[] = "Usage: %s [optins]\n"
                   "Options are:\n"
                   "    -i   unshare IPC namespace\n"
                   "    -m   unshare mount namespace\n"
                   "    -n   unshare network namespace\n"
                   "    -p   unshare PID namespace\n"
                   "    -u   unshare UTS namespace\n"
                   "    -U   unshare user namespace\n";
    printf(usage, pname);
    exit(0);
}

int main(int argc, char *argv[])
{
    int flags = 0, opt, ret;

    while ((opt = getopt(argc, argv, "imnpuUh")) != -1) {
        switch (opt) {
            case 'i': flags |= CLONE_NEWIPC;        break;
            case 'm': flags |= CLONE_NEWNS;         break;
            case 'n': flags |= CLONE_NEWNET;        break;
            case 'p': flags |= CLONE_NEWPID;        break;
            case 'u': flags |= CLONE_NEWUTS;        break;
            case 'U': flags |= CLONE_NEWUSER;       break;
            case 'h': usage(argv[0]);               break;
            default:  usage(argv[0]);
        }
    }

    if (flags == 0) {
        usage(argv[0]);
    }

    //执行完unshare函数后，当前进程就会退出当前的一个或多个类型的namespace,
    //然后进入到一个或多个新创建的不同类型的namespace
    ret = unshare(flags);
    NOT_OK_EXIT(ret, "unshare");

    //新的bash进程会继承当前进程的所有namespace
    system("/bin/bash");

    return 0;
}
```
这里没有包含CLONE_NEWCGROUP，原因是因为ubuntu 16.04对cgroup namespace支持的还不够完善，相关头文件中不存在这个宏的定义。

看运行效果：
```bash
#将上面的代码保存为文件namespace_leave.c并编译
dev@ubuntu:~/code$ gcc namespace_leave.c -o namespace_leave

#查看当前bash所属的UTS namespace
dev@ubuntu:~/code$ sudo readlink /proc/$$/ns/uts
uts:[4026531838]

#执行程序
dev@ubuntu:~/code$ sudo ./namespace_leave -u
root@ubuntu:~/code#

#再次查看UTS namespace，已经变了，说明已经离开原来的namespace并加入了新的namespace
root@ubuntu:~/code# readlink /proc/$$/ns/uts
uts:[4026532456]

#反复执行几次，得到同样的结果
root@ubuntu:~/code# ./namespace_leave -u
root@ubuntu:~/code# readlink /proc/$$/ns/uts
uts:[4026532457]
root@ubuntu:~/code# ./namespace_leave -u
root@ubuntu:~/code# readlink /proc/$$/ns/uts
uts:[4026532458]
root@ubuntu:~/code# ./namespace_leave -u
root@ubuntu:~/code# readlink /proc/$$/ns/uts
uts:[4026532459]
```
###内核中的实现
在老版本中，UTS相关的信息保存在一个全局变量中，所有进程都共享这个全局变量，gethostname()的实现大概如下
```c
asmlinkage long sys_gethostname(char __user *name, int len)
{
  ...
  if (copy_to_user(name, system_utsname.nodename, i))
    errno = -EFAULT;
  ...
}
```

在新的Linux内核中，在每个进程对应的task结构体[struct task_struct](https://github.com/torvalds/linux/blob/master/include/linux/sched.h)中，增加了一个叫nsproxy的字段，类型是[struct nsproxy](https://github.com/torvalds/linux/blob/master/include/linux/nsproxy.h)
```c
struct task_struct {
  ...
  /* namespaces */
  struct nsproxy *nsproxy;
  ...
}

struct nsproxy {
  atomic_t count;
  struct uts_namespace *uts_ns;
  struct ipc_namespace *ipc_ns;
  struct mnt_namespace *mnt_ns;
  struct pid_namespace *pid_ns_for_children;
  struct net       *net_ns;
  struct cgroup_namespace *cgroup_ns;
};
```

于是新的gethostname()的实现大概就是这样
```c
static inline struct new_utsname *utsname(void)
{
  return &current->nsproxy->uts_ns->name;
}

SYSCALL_DEFINE2(gethostname, char __user *, name, int, len)
{
  struct new_utsname *u;
  ...
  u = utsname();
  if (copy_to_user(name, u->nodename, i)){
    errno = -EFAULT;
  }
  ...
}
```
处于不同UTS namespace中的进程，它task结构体里面的nsproxy->uts_ns所指向的结构体是不一样的，于是达到了隔离UTS的目的。

其他类型的namespace基本上也是差不多的原理。

###总结
* namespace的本质就是把原来所有进程全局共享的资源拆分成了很多个一组一组进程共享的资源
* UTS namespace就是进程的一个属性，属性值相同的一组进程就属于同一个namespace，跟这组进程之间有没有亲戚关系无关
* clone和unshare都有创建并加入新的namespace的功能，他们的主要区别是：
    * unshare是使当前进程加入新创建的namespace
    * clone是创建一个新的子进程，然后让子进程加入新的namespace
* UTS namespace没有嵌套关系，即不存在说一个namespace是另一个namespace的父namespace，也不存在相应的函数能隐式的实现子进程退出当前namespace而回到父进程的namespace，只能通过setns来显式的加入父进程所在的namespace

上面的总结基本上也适用于其他类型的namespace，有些特殊的情况会在介绍具体类型的namespace时提到。

###参考

* [Namespaces in operation, part 2: the namespaces API](https://lwn.net/Articles/531114/)
* [Resource management:Linux kernel Namespaces and cgroups](http://www.haifux.org/lectures/299/netLec7.pdf)