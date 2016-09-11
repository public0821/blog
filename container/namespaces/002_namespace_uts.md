# Namespace系列（02）：UTS namespace (CLONE_NEWUTS)
UTS namespace用来隔离系统的hostname以及NIS domain name。

这两个资源可以通过sethostname(2) and setdomainname(2)函数来设置，以及通过uname(2), gethostname(2), and getdomainname(2)函数来获取.（这里括号中的2表示这个函数是system call，具体其他数字的含义请参看man的帮助文件）

术语UTS来自于调用函数uname()时用到的结构体: struct utsname. 而这个结构体的名字源自于"UNIX Time-sharing System".

由于UTS namespace最简单，所以放在最前面介绍，在这篇文章中我们将会熟悉UTS  namespace以及和namespace相关的三个系统调用的使用。

>注意： NIS domain name和DNS没有关系，关于他的介绍可以看[这里](https://www.freebsd.org/doc/handbook/network-nis.html)，由于本人对它不了解，所以在本文中不做介绍。

##创建新的UTS namespace
多说无益，直接上代码，我尽量将注释写的足够详细，请仔细看代码和输出结果

>**注意:**
1. 为了代码简单起见，只在clone函数那做了错误处理，关于clone函数的详细介绍请参考[man-pages](http://man7.org/linux/man-pages/man2/clone.2.html)
2. 为了描述方便，某些地方会用hostname来区分UTS namespace，如hostname为container001的namespace，我们会描述成namespace container001。

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
static int child_func(void *hostname)
{
    //设置主机名
    sethostname(hostname, strlen(hostname));

    //用一个新的bash来替换掉当前子进程，
    //执行完execlp后，子进程没有退出，也没有创建新的进程,
    //而是当前子进程不再运行自己的代码，而是去执行bash的代码,
    //详情请参考"man execlp"
    //bash退出后，子进程执行完毕
    execlp("bash", "bash", (char *) NULL);

    //从这里开始的代码将不会被执行到，因为当前子进程已经被上面的bash替换掉了

    return 0;
}

static char child_stack[1024*1024]; //设置子进程的栈空间为1M

int main(int argc, char *argv[])
{
    pid_t child_pid;

    if (argc < 2) {
        printf("Usage: %s <child-hostname>\n", argv[0]);
        return -1;
    }

    //创建并启动子进程，调用该函数后，父进程将继续往后执行，也就是执行后面的waitpid
    child_pid = clone(child_func,  //子进程将执行child_func这个函数
                    //栈是从高位向低位增长，所以这里要指向高位地址
                    child_stack + sizeof(child_stack),
                    //CLONE_NEWUTS表示创建新的UTS namespace，
                    //这里SIGCHLD是子进程退出后返回给父进程的信号，跟namespace无关
                    CLONE_NEWUTS | SIGCHLD,
                    argv[1]);  //传给child_func的参数
    NOT_OK_EXIT(child_pid, "clone");

    waitpid(child_pid, NULL, 0); //等待子进程结束

    return 0;    //这行执行完之后，父进程结束
}
```

在上面的代码中：

* 父进程创建新的子进程，并且设置CLONE_NEWUTS，这样就会创建新的UTS namespace并且让子进程属于这个新的namespace，然后父进程一直等待子进程退出
* 子进程在设置好新的hostname后被bash替换掉
* 当bash退出后，子进程退出，接着父进程也退出

下面看看输出效果
```bash
#------------------------第一个shell窗口------------------------
#将上面的代码保持为namespace_uts_demo.c， 
#然后用gcc将它编译成可执行文件namespace_uts_demo
dev@ubuntu:~/code$ gcc namespace_uts_demo.c -o namespace_uts_demo   

#启动程序，传入参数container001
#创建新的UTS namespace需要root权限，所以用到sudo
dev@ubuntu:~/code$ sudo ./namespace_uts_demo container001

#新的bash被启动，从shell的提示符可以看出，hostname已经被改成了container001
#这里bash的提示符是‘#’，表示bash有root权限，
#这是因为我们是用sudo来运行的程序，于是我们程序创建的子进程有root权限
root@container001:~/code#

#用hostname命令再确认一下
root@container001:~/code# hostname
container001

#pstree是用来查看系统中进程之间父子关系的工具
#下面的输出过滤掉了跟namespace_uts_demo无关的内容
#本次操作是通过ssh客户端远程连接到Linux主机进行的，
#所以bash(24429)的父进程是一系列的sshd进程，
#我们在bash(24429)里面执行了sudo ./namespace_uts_demo container001
#所以有了sudo(27332)和我们程序namespace_uts_d(27333)对应的进程，
#我们的程序自己clone了一个新的子进程，由于clone的时候指定了参数CLONE_NEWUTS，
#所以新的子进程属于一个新的UTS namespace，然后这个新进程调用execlp后被bash替换掉了，
#于是有了bash(27334)， 这个bash进程拥有所有当前子进程的属性， 
#由于我们的pstree命令是在bash(27334)里面运行的，
#所以这里pstree(27345)是bash(27334)的子进程
root@container001:~/code# pstree -pl
systemd(1)───sshd(24351)───sshd(24428)───bash(24429)───sudo(27332)──
─namespace_uts_d(27333)───bash(27334)───pstree(27345)

#验证一下我们运行的bash进程是不是bash(27334)
#下面这个命令可以输出当前bash的PID
root@container001:~/code# echo $$
27334

#验证一下我们的父进程和子进程是否不在同一个UTS namespace
root@container001:~/code# readlink /proc/27333/ns/uts
uts:[4026531838]
root@container001:~/code# readlink /proc/27334/ns/uts
uts:[4026532445]
#果然不属于同一个UTS namespace，说明新的uts namespace创建成功

#默认情况下，子进程应该继承父进程的namespace
#systemd(1)是我们程序父进程namespace_uts_d(27333)的祖先进程，
#他们应该属于同一个namespace
root@container001:~/code# readlink /proc/1/ns/uts
uts:[4026531838]

#所有bash(27334)里面执行的进程应该和bash(27334)属于同样的namespace
#self指向当前运行的进程，在这里即readlink进程
root@container001:~/code# readlink /proc/self/ns/uts
uts:[4026532445]

#------------------------第二个shell窗口------------------------
#重新打开一个新的shell窗口，确认这个shell和上面的namespace_uts_d(27333)属于同一个namespace
dev@ubuntu:~/code$ readlink /proc/$$/ns/uts
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

#进程间的关系和上面的差不多，在后面又生成了namespace_uts_d(27354)和bash(27355)
root@container002:~/code# pstree -pl
systemd(1)───sshd(24351)───sshd(24428)───bash(24429)───sudo(27332)──
─namespace_uts_d(27333)───bash(27334)───namespace_uts_d(27354)──
─bash(27355)───pstree(27367)

#退出bash(27355)后，它的父进程namespace_uts_d(27354)也接着退出，
#于是又回到了进程bash(27334)中，hostname于是也回到了container001
#注意： 在bash(27355)退出的过程中，并没有任何进程的namespace发生变化，
#只是所有属于namespace container002的进程都执行完退出了
root@container002:~/code# exit
exit
root@container001:~/code#
root@container001:~/code# hostname
container001
```

##将当前进程加入指定的namespace
还是直接上代码，有了前面的铺垫，这里的代码就非常简单了，请仔细看代码和输出结果
```c
#define _GNU_SOURCE
#include <fcntl.h>
#include <sched.h>
#include <stdlib.h>
#include <stdio.h>
#include <unistd.h>

#define NOT_OK_EXIT(code, msg); {if(code == -1){perror(msg); exit(-1);} }

int main(int argc, char *argv[])
{
    int fd, ret;

    if (argc < 2) {
        printf("%s /proc/PID/ns/FILE\n", argv[0]);
        return -1;
    }

    //获取namespace对应文件的描述符
    fd = open(argv[1], O_RDONLY);
    NOT_OK_EXIT(fd, "open");

    //执行完setns后，当前进程将加入指定的namespace
    //这里第二个参数为0，表示由系统自己检测fd对应的是哪种类型的namespace
    ret = setns(fd, 0);
    NOT_OK_EXIT(ret, "open");

    //用一个新的bash来替换掉当前子进程
    execlp("bash", "bash", (char *) NULL);

    return 0;
}

```
在上面的代码中，程序通过setns调用让自己加入到参数指定的namespace中，然后用bash替换掉自己，开始执行bash。

再来看结果
```bash
#--------------------------第一个shell窗口----------------------
#重用上面创建的namespace container001
#先确认一下hostname是否正确，
root@container001:~/code# hostname
container001

#获取bash的PID
root@container001:~/code# echo $$
27334

#得到bash所属的UTS namespace
root@container001:~/code# readlink /proc/27334/ns/uts
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
#27334是第一个shell窗口中bash的pid
dev@ubuntu:~/code$ sudo ./namespace_join /proc/27334/ns/uts
root@container001:~/code#

#加入成功，bash的hostname和UTS namespace的number和第一个shell窗口的都一样
root@container001:~/code# hostname
container001
root@container001:~/code# readlink /proc/$$/ns/uts
uts:[4026532445]
```

##退出当前namespace并加入新创建的namespace
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

    //解析命令行参数，用来决定退出哪个类型的namespace
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

    //用一个新的bash来替换掉当前子进程
    execlp("bash", "bash", (char *) NULL);

    return 0;
}
```

看运行效果：
```bash
#将上面的代码保存为文件namespace_leave.c并编译
dev@ubuntu:~/code$ gcc namespace_leave.c -o namespace_leave

#查看当前bash所属的UTS namespace
dev@ubuntu:~/code$ readlink /proc/$$/ns/uts
uts:[4026531838]

#执行程序， -u表示退出并加入新的UTS namespace
dev@ubuntu:~/code$ sudo ./namespace_leave -u
root@ubuntu:~/code#

#再次查看UTS namespace，已经变了，说明已经离开原来的namespace并加入了新的namespace
#细心的同学可能已经发现这里的inode number刚好和上面namespace container002的相同，
#这说明在container002被销毁后，inode number被回收再利用了
root@ubuntu:~/code# readlink /proc/$$/ns/uts
uts:[4026532455]

#反复执行几次，得到类似的结果
root@ubuntu:~/code# ./namespace_leave -u
root@ubuntu:~/code# readlink /proc/$$/ns/uts
uts:[4026532456]
root@ubuntu:~/code# ./namespace_leave -u
root@ubuntu:~/code# readlink /proc/$$/ns/uts
uts:[4026532457]
root@ubuntu:~/code# ./namespace_leave -u
root@ubuntu:~/code# readlink /proc/$$/ns/uts
uts:[4026532458]
```

##内核中的实现
上面演示了这三个函数的功能，那么UTS namespace在内核中又是怎么实现的呢？

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
  //current指向当前进程的task结构体
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

##总结
* namespace的本质就是把原来所有进程全局共享的资源拆分成了很多个一组一组进程共享的资源
* 当一个namespace里面的所有进程都退出时，namespace也会被销毁，所以抛开进程谈namespace没有意义
* UTS namespace就是进程的一个属性，属性值相同的一组进程就属于同一个namespace，跟这组进程之间有没有亲戚关系无关
* clone和unshare都有创建并加入新的namespace的功能，他们的主要区别是：
    * unshare是使当前进程加入新创建的namespace
    * clone是创建一个新的子进程，然后让子进程加入新的namespace
* UTS namespace没有嵌套关系，即不存在说一个namespace是另一个namespace的父namespace


##参考

* [Namespaces in operation, part 2: the namespaces API](https://lwn.net/Articles/531114/)
* [Resource management:Linux kernel Namespaces and cgroups](http://www.haifux.org/lectures/299/netLec7.pdf)