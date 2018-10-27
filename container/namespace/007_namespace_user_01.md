# Linux Namespace系列（07）：user namespace (CLONE_NEWUSER)  (第一部分)
User namespace用来隔离user权限相关的Linux资源，包括[user IDs and group IDs](http://man7.org/linux/man-pages/man7/credentials.7.html)，[keys](http://man7.org/linux/man-pages/man2/keyctl.2.html) , 和[capabilities](http://man7.org/linux/man-pages/man7/capabilities.7.html). 

这是目前实现的namespace中最复杂的一个，因为user和权限息息相关，而权限又事关容器的安全，所以稍有不慎，就会出安全问题。

user namespace可以嵌套（目前内核控制最多32层），除了系统默认的user namespace外，所有的user namespace都有一个父user namespace，每个user namespace都可以有零到多个子user namespace。 当在一个进程中调用unshare或者clone创建新的user namespace时，当前进程原来所在的user namespace为父user namespace，新的user namespace为子user namespace.

在不同的user namespace中，同样一个用户的user ID 和group ID可以不一样，换句话说，一个用户可以在父user namespace中是普通用户，在子user namespace中是超级用户（超级用户只相对于子user namespace所拥有的资源，无法访问其他user namespace中需要超级用户才能访问资源）。

从Linux 3.8开始，创建新的user namespace不需要root权限。

>本篇所有例子都在ubuntu-server-x86_64 16.04下执行通过

## 创建user namespace
```bash
#--------------------------第一个shell窗口----------------------
#先记录下目前的id，gid和user namespace
dev@ubuntu:~$ id
uid=1000(dev) gid=1000(dev) groups=1000(dev),4(adm),24(cdrom),27(sudo)
dev@ubuntu:~$ readlink /proc/$$/ns/user
user:[4026531837]

#创建新的user namespace
dev@ubuntu:~$ unshare --user /bin/bash
nobody@ubuntu:~$ readlink /proc/$$/ns/user
user:[4026532464]
nobody@ubuntu:~$ id
uid=65534(nobody) gid=65534(nogroup) groups=65534(nogroup)
```

很奇怪，为什么上面例子中显示的用户名是nobody，它的id和gid都是65534？

这是因为我们还没有映射父user namespace的user ID和group ID到子user namespace中来，这一步是必须的，因为这样系统才能控制一个user namespace里的用户在其他user namespace中的权限。（比如给其他user namespace中的进程发送信号，或者访问属于其他user namespace挂载的文件）

如果没有映射的话，当在新的user namespace中用getuid()和getgid()获取user id和group id时，系统将返回文件/proc/sys/kernel/overflowuid中定义的user ID以及proc/sys/kernel/overflowgid中定义的group ID，它们的默认值都是65534。也就是说如果没有指定映射关系的话，会默认映射到ID65534。

下面看看这个user能干些什么
```bash
#--------------------------第一个shell窗口----------------------
#ls的结果显示/root目录属于nobody
nobody@ubuntu:~$ ls -l /|grep root
drwx------   3 nobody nogroup  4096 7月   8 18:39 root
#但是当前的nobody账号访问不了，说明这两个nobody不是一个ID，他们之间没有映射关系
nobody@ubuntu:~$ ls /root
ls: cannot open directory '/root': Permission denied

#这里显示/home/dev目录属于nobody
nobody@ubuntu:~$ ls -l /home/
drwxr-xr-x 11 nobody nogroup 4096 7月   8 18:40 dev
#touch成功，说明虽然没有显式的映射ID，但还是能访问父user namespace里dev账号拥有的资源
#说明他们背后还是有映射关系
nobody@ubuntu:~$ touch /home/dev/temp01
nobody@ubuntu:~$
```

## 映射user ID和group ID
通常情况下，创建新的user namespace后，第一件事就是映射user和group ID. 映射ID的方法是添加配置到/proc/PID/uid_map和/proc/PID/gid_map（这里的PID是新user namespace中的进程ID，刚开始时这两个文件都是空的）. 

这两个文件里面的配置格式如下（可以有多条）：
```    
    ID-inside-ns   ID-outside-ns   length
```

举个例子, 0 1000 256这条配置就表示父user namespace中的1000~1256映射到新user namespace中的0~256。 



>系统默认的user namespace没有父user namespace，但为了保持一致，kernel提供了一个虚拟的uid和gid map文件，看起来是这样子的:
>dev@ubuntu:~$ cat /proc/$$/uid_map
>0          0 4294967295
 

那么谁可以向这个文件中写配置呢？

/proc/PID/uid_map和/proc/PID/gid_map的拥有者是创建新user namespace的这个user，所以和这个user在一个user namespace的root账号可以写。但这个user自己有没有写map文件权限还要看它有没有CAP_SETUID和CAP_SETGID的capability。

>**注意**：只能向map文件写一次数据，但可以一次写多条，并且最多只能5条

关于capability的详细介绍可以参考[这里](http://man7.org/linux/man-pages/man7/capabilities.7.html)，简单点说，原来的Linux就分root和非root，很多操作只能root完成，比如修改一个文件的owner，后来Linux将root的一些权限分解了，变成了各种capability，只要拥有了相应的capability，就能做相应的操作，不需要root账户的权限。

下面我们来看看如何用dev账号映射uid和gid
```bash
#--------------------------第一个shell窗口----------------------
#获取当前bash的pid
nobody@ubuntu:~$ echo $$
24126

#--------------------------第二个shell窗口----------------------
#dev是map文件的owner
dev@ubuntu:~$ ls -l /proc/24126/uid_map /proc/24126/gid_map
-rw-r--r-- 1 dev dev 0 7月  24 23:11 /proc/24126/gid_map
-rw-r--r-- 1 dev dev 0 7月  24 23:11 /proc/24126/uid_map

#但还是没有权限写这个文件
dev@ubuntu:~$ echo '0 1000 100' > /proc/24126/uid_map
bash: echo: write error: Operation not permitted
dev@ubuntu:~$ echo '0 1000 100' > /proc/24126/gid_map
bash: echo: write error: Operation not permitted
#当前用户运行的bash进程没有CAP_SETUID和CAP_SETGID的权限
dev@ubuntu:~$ cat /proc/$$/status | egrep 'Cap(Inh|Prm|Eff)'
CapInh: 0000000000000000
CapPrm: 0000000000000000
CapEff: 0000000000000000

#为/binb/bash设置capability，
dev@ubuntu:~$ sudo setcap cap_setgid,cap_setuid+ep /bin/bash
#重新加载bash以后我们看到相应的capability已经有了
dev@ubuntu:~$ exec bash
dev@ubuntu:~$ cat /proc/$$/status | egrep 'Cap(Inh|Prm|Eff)'
CapInh: 0000000000000000
CapPrm: 00000000000000c0
CapEff: 00000000000000c0


#再试一次写map文件，成功了
dev@ubuntu:~$ echo '0 1000 100' > /proc/24126/uid_map
dev@ubuntu:~$ echo '0 1000 100' > /proc/24126/gid_map
dev@ubuntu:~$
#再写一次就失败了，因为这个文件只能写一次
dev@ubuntu:~$ echo '0 1000 100' > /proc/24126/uid_map
bash: echo: write error: Operation not permitted
dev@ubuntu:~$ echo '0 1000 100' > /proc/24126/gid_map
bash: echo: write error: Operation not permitted

#后续测试不需要CAP_SETUID了，将/bin/bash的capability恢复到原来的设置
dev@ubuntu:~$ sudo setcap cap_setgid,cap_setuid-ep /bin/bash
dev@ubuntu:~$ getcap /bin/bash
/bin/bash =

#--------------------------第一个shell窗口----------------------
#回到第一个窗口，id已经变成0了，说明映射成功
nobody@ubuntu:~$ id
uid=0(root) gid=0(root) groups=0(root),65534(nogroup)

#--------------------------第二个shell窗口----------------------
#回到第二个窗口，确认map文件的owner，这里24126是新user namespace中的bash
dev@ubuntu:~$ ls -l /proc/24126/
......
-rw-r--r-- 1 dev dev 0 7月  24 23:13 gid_map
dr-x--x--x 2 dev dev 0 7月  24 23:10 ns
-rw-r--r-- 1 dev dev 0 7月  24 23:13 uid_map
......

#--------------------------第一个shell窗口----------------------
#重新加载bash，提示有root权限了
nobody@ubuntu:~$ exec bash
root@ubuntu:~#

#0000003fffffffff表示当前运行的bash拥有所有的capability
root@ubuntu:~# cat /proc/$$/status | egrep 'Cap(Inh|Prm|Eff)'
CapInh: 0000000000000000
CapPrm: 0000003fffffffff
CapEff: 0000003fffffffff
#--------------------------第二个shell窗口----------------------
#回到第二个窗口，发现owner已经变了，变成了root
#目前还不清楚为什么有这样的机制
dev@ubuntu:~$ ls -l /proc/24126/
......
-rw-r--r-- 1 root root 0 7月  24 23:13 gid_map
dr-x--x--x 2 root root 0 7月  24 23:10 ns
-rw-r--r-- 1 root root 0 7月  24 23:13 uid_map
......
#虽然不能看目录里有哪些文件，但是可以读里面文件的内容
dev@ubuntu:~$ ls -l /proc/24126/ns
ls: cannot open directory '/proc/24126/ns': Permission denied
dev@ubuntu:~$ readlink /proc/24126/ns/user
user:[4026532464]


#--------------------------第一个shell窗口----------------------
#和第二个窗口一样的结果
root@ubuntu:~# ls -l /proc/24126/ns
ls: cannot open directory '/proc/24126/ns': Permission denied
root@ubuntu:~# readlink /proc/24126/ns/user
user:[4026532464]

#仍然不能访问/root目录，因为他的拥有着是nobody
root@ubuntu:~# ls -l /|grep root
drwx------   3 nobody nogroup  4096 7月   8 18:39 root
root@ubuntu:~# ls /root
ls: cannot open directory '/root': Permission denied

#对于原来/home/dev下的内容，显示的owner已经映射过来了，由dev变成了新namespace中的root，
#当前root用户可以访问他里面的内容
root@ubuntu:~# ls -l /home
drwxr-xr-x 8 root root 4096 7月  21 18:35 dev
root@ubuntu:~# touch /home/dev/temp01
root@ubuntu:~#

#试试设置主机名称
root@ubuntu:~# hostname container001
hostname: you must be root to change the host name
#修改失败，说明这个新user namespace中的root账号在父user namespace里面不好使
#这也正是user namespace所期望达到的效果，当访问其他user namespace里的资源时，
#是以其他user namespace中的相应账号的权限来执行的，
#比如这里root对应父user namespace的账号是dev，所以改不了系统的hostname
```

那是不是把系统默认user namespace的root账号映射到新的user namespace中，新user namespace的root就可以修改默认user namespace中的hostname呢？
```bash
#--------------------------第三个shell窗口----------------------
#重新打开一个窗口
#这里不再手动映射uid和gid，而是利用unshare命令的-r参数来帮我们完成映射，
#指定-r参数后，unshare将会帮助我们将当前运行unshare的账号映射成新user namesapce的root账号
#这里用了sudo，目的是让root账号来运行unshare命令，
#这样就将外面的root账号映射成新user namespace的root账号
dev@ubuntu:~$ sudo unshare --user -r /bin/bash
root@ubuntu:~# id
uid=0(root) gid=0(root) groups=0(root)

#确认是用root映射root
root@ubuntu:~# echo $$
24283
root@ubuntu:~# cat /proc/24283/uid_map
         0          0          1
root@ubuntu:~# cat /proc/24283/gid_map
         0          0          1

#可以访问/root目录下的东西，但无法操作/home/dev/下的文件
root@ubuntu:~# ls -l / |grep root$
drwx------   6 root root  4096 8月  14 23:11 root
root@ubuntu:~# touch /root/temp01
root@ubuntu:~# ls -l /home
drwxr-xr-x 11 nobody nogroup 4096 8月  14 23:13 dev
root@ubuntu:~# touch /home/dev/temp01
touch: cannot touch '/home/dev/temp01': Permission denied

#尝试修改hostname，还是失败
root@ubuntu:~# hostname container001
hostname: you must be root to change the host name
```

上面的例子中虽然是将root账号映射到了新user namespace的root账号上，但修改hostname、访问/home/dev下的文件依然失败，那是因为不管怎么映射，当用子user namespace的账号访问父user namespace的资源的时候，它启动的进程的capability都为空，所以这里子user namespace的root账号到父namespace中就相当于一个普通的账号。

**注意：**对于map文件来说，在父user namespace和子user namespac中打开子user namespace中进程的这个文件看到的都是同样的内容，但如果是在其他的user namespace中打开这个map文件，‘ID-outside-ns’表示的就是映射到当前user namespace的ID.这里听起来有点绕，看下面的例子

```bash
#--------------------------打开一个新窗口----------------------
#创建一个新的user namespace，并取名container001
dev@ubuntu:~$ unshare --user --uts -r /bin/bash
root@ubuntu:~# hostname container001
root@ubuntu:~# exec bash
#记下bash的pid
root@container001:~# echo $$
27898

#在container001里面创建新的namespace container002
root@container001:~# unshare --user --uts -r /bin/bash
root@container001:~# hostname container002
root@container001:~# exec bash
#记下bash的pid
root@container002:~# echo $$
28066

#查看自己namespace中进程的uid map文件
#这里表示父user namespace的0映射到了当前namespace的0
root@container002:~# cat /proc/28066/uid_map
         0          0          1

#--------------------------再打开一个新窗口----------------------
#在系统默认namespace中查看同样这个文件，发现和上面的显示的不一样
#因为默认namespace是container002的爷爷，所以他们两个里面看到的东西有可能不一样
#这里表示当前user namespace的账号1000映射到了进程28066所在user namespace的账号0
#当然如果上面是用root账号创建的container001，这里显示的内容就和上面一样了
dev@ubuntu:~$ cat /proc/28066/uid_map
         0       1000          1

#我们再进入到container001，在里面看看这个文件，发现和在ontainer002看到的结果一样
#说明对于进程28066来说，在他自己所在的user namespace和他的父user namespace看到的map文件内容是一样的
dev@ubuntu:~$ nsenter --user --uts -t 27898 --preserve-credentials bash
root@container001:~# cat /proc/28066/uid_map
         0          0          1
#默认情况下，nsenter会调用setgroups函数去掉root group的权限，
#这里--preserve-credentials是为了让nsenter不调用setgroups函数，因为调用这个函数需要root权限

#测试完成后可以关闭这两个窗口，后面不会再用到了
```

## user namespace的owner
当一个用户创建一个新的user namespace的时候，这个用户就是这个新user namespace的owner，在父user namespace的这个用户就会拥有新user namespace及其所有子孙user namespace的所有capabilities. 
```bash
#--------------------------第四个shell窗口----------------------
#新建用户test用于测试
dev@ubuntu:~$ sudo useradd test
dev@ubuntu:~$ sudo passwd test
Enter new UNIX password:
Retype new UNIX password:
passwd: password updated successfully

#切换到test账户并创建新的user namespace
#为了便于区分，同时创建新的uts namespace
dev@ubuntu:~$ su test
Password:
test@ubuntu:/home/dev$ unshare --user --uts -r /bin/bash

#设置一个容易区分的hostname
root@ubuntu:/home/dev# hostname container001
root@ubuntu:/home/dev# exec bash
root@container001:/home/dev# readlink /proc/$$/ns/user
user:[4026532463]
root@container001:/home/dev# echo $$
24419

#--------------------------第五个shell窗口----------------------
#使用dev账号新建一个user namespace
dev@ubuntu:~$ unshare --user --uts -r /bin/bash
root@ubuntu:~# hostname container002
root@ubuntu:~# exec bash
root@container002:~# readlink /proc/$$/ns/user
user:[4026532464]
root@container002:~# echo $$
24435

#--------------------------第六个shell窗口----------------------
#用dev账号往container002中加入新的进程/bin/bash成功，因为dev是container002的owner
dev@ubuntu:~$ nsenter --user -t 24435 --preserve-credentials --uts /bin/bash
root@container002:~# id
uid=0(root) gid=0(root) groups=0(root),65534(nogroup)
root@container002:~# readlink /proc/$$/ns/user
user:[4026532464]

#回到默认user namespace
root@container002:~# exit
exit
dev@ubuntu:~$

#因为container001的owner是test，用dev账号往container001中加入新的进程/bin/bash失败
dev@ubuntu:~$ nsenter --user -t 24419 --preserve-credentials --uts /bin/bash
nsenter: cannot open /proc/24419/ns/user: Permission denied

#用root账号往container001中加入新的进程/bin/bash成功
dev@ubuntu:~$ sudo nsenter --user -t 24419 --preserve-credentials --uts /bin/bash
nobody@container001:~$ readlink /proc/$$/ns/user
user:[4026532463]
#由于root账号没有映射到container001中，所以这里在container001中看到的账号是nobody
nobody@container001:~$ id
uid=65534(nobody) gid=65534(nogroup) groups=65534(nogroup)

#退出container001，便于后续测试
nobody@container001:~$ exit
dev@ubuntu:~$
#--------------------------第五个shell窗口----------------------
#回到第5个窗口，继续创建一个新的user namespace
root@container002:~# unshare --user --uts -r /bin/bash
root@container002:~# hostname container003
root@container002:~# exec bash
root@container003:~# readlink /proc/$$/ns/user
user:[4026532471]
root@container003:~# echo $$
24533

#--------------------------第六个shell窗口----------------------
#回到第6个窗口，用dev账号往container003（孙子user namespace）中加入新的bash进程，成功，
#说明dev拥有孙子user namespace的capabilities
dev@ubuntu:~$ nsenter --user -t 24533 --preserve-credentials --uts /bin/bash
root@container003:~# readlink /proc/$$/ns/user
user:[4026532471]

```

## 结束语
本文先介绍了user namespace的一些概念，然后介绍如何配置mapping文件，最后介绍了user namespace的owner。从上面的介绍中可以看出，user namespace还是比较复杂的，要了解user namespace，需要对Linux下的权限有一个基本的了解。下一篇中将继续介绍user namespace和其他namespace的关系，以及一些其他的注意事项。

## 参考
* [user namespaces man page](http://man7.org/linux/man-pages/man7/user_namespaces.7.html)
* [Namespaces in operation, part 5: User namespaces](https://lwn.net/Articles/532593/)
* [Namespaces in operation, part 6: more on user namespaces](https://lwn.net/Articles/540087/)
