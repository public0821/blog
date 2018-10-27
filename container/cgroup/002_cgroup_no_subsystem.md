# Linux Cgroup系列（02）：创建并管理cgroup

本文将创建并挂载一颗不和任何subsystem绑定的cgroup树，用来演示怎么创建、删除子cgroup，以及如何往cgroup中添加和删除进程。

由于不和任何subsystem绑定，所以这棵树没有任何实际的功能，但这不影响我们的演示，还有一个好处就是我们不会受subsystem功能的影响，可以将精力集中在cgroup树上。

>本篇所有例子都在ubuntu-server-x86_64 16.04下执行通过

## 挂载cgroup树
开始使用cgroup前需要先挂载cgroup树，下面先看看如何挂载一颗cgroup树，然后再查看其根目录下生成的文件
```bash
#准备需要的目录
dev@ubuntu:~$ mkdir cgroup && cd cgroup
dev@ubuntu:~/cgroup$ mkdir demo

#由于name=demo的cgroup树不存在，所以系统会创建一颗新的cgroup树，然后挂载到demo目录
dev@ubuntu:~/cgroup$ sudo mount -t cgroup -o none,name=demo demo ./demo

#挂载点所在目录就是这颗cgroup树的root cgroup，在root cgroup下面，系统生成了一些默认文件
dev@ubuntu:~/cgroup$ ls ./demo/
cgroup.clone_children  cgroup.procs  cgroup.sane_behavior  notify_on_release  release_agent  tasks

#cgroup.procs里包含系统中的所有进程
dev@ubuntu:~/cgroup$ wc -l ./demo/cgroup.procs
131 ./demo/cgroup.procs

```
下面是每个文件的含义：

* cgroup.clone_children
这个文件只对cpuset（subsystem）有影响，当该文件的内容为1时，新创建的cgroup将会继承父cgroup的配置，即从父cgroup里面拷贝配置文件来初始化新cgroup，可以参考[这里](https://lkml.org/lkml/2010/7/29/368)

* cgroup.procs
当前cgroup中的所有进程ID，系统不保证ID是顺序排列的，且ID有可能重复

* cgroup.sane_behavior
具体功能不详，可以参考[这里](https://lkml.org/lkml/2014/7/2/684)和[这里](https://lkml.org/lkml/2014/7/2/686)

* notify_on_release
该文件的内容为1时，当cgroup退出时（不再包含任何进程和子cgroup），将调用release_agent里面配置的命令。新cgroup被创建时将默认继承父cgroup的这项配置。

* release_agent
里面包含了cgroup退出时将会执行的命令，系统调用该命令时会将相应cgroup的相对路径当作参数传进去。 注意：这个文件只会存在于root cgroup下面，其他cgroup里面不会有这个文件。

* tasks
当前cgroup中的所有线程ID，系统不保证ID是顺序排列的

后面在介绍如何往cgroup中添加进程时会介绍cgroup.procs和tasks的差别。

## 创建和删除cgroup
挂载好上面的cgroup树之后，就可以在里面建子cgroup了
```bash
#创建子cgroup很简单，新建一个目录就可以了
dev@ubuntu:~/cgroup$ cd demo
dev@ubuntu:~/cgroup/demo$ sudo mkdir cgroup1

#在新创建的cgroup里面，系统默认也生成了一些文件，这些文件的意义和root cgroup里面的一样
dev@ubuntu:~/cgroup/demo$ ls cgroup1/
cgroup.clone_children  cgroup.procs  notify_on_release  tasks

#新创建的cgroup里没有任何进程和线程
dev@ubuntu:~/cgroup/demo$ wc -l cgroup1/cgroup.procs
0 cgroup1/cgroup.procs
dev@ubuntu:~/cgroup/demo$ wc -l cgroup1/tasks
0 cgroup1/tasks

#每个cgroup都可以创建自己的子cgroup，所以我们也可以在cgroup1里面创建子cgroup
dev@ubuntu:~/cgroup/demo$ sudo mkdir cgroup1/cgroup11
dev@ubuntu:~/cgroup/demo$ ls cgroup1/cgroup11
cgroup.clone_children  cgroup.procs  notify_on_release  tasks

#删除cgroup也很简单，删除掉相应的目录就可以了
dev@ubuntu:~/cgroup/demo$ sudo rmdir cgroup1/
rmdir: failed to remove 'cgroup1/': Device or resource busy
#这里删除cgroup1失败，是因为它里面包含了子cgroup，所以不能删除，
#如果cgroup1包含有进程或者线程，也会删除失败

#先删除cgroup11，再删除cgroup1就可以了
dev@ubuntu:~/cgroup/demo$ sudo rmdir cgroup1/cgroup11/
dev@ubuntu:~/cgroup/demo$ sudo rmdir cgroup1/
```

## 添加进程
创建新的cgroup后，就可以往里面添加进程了。注意下面几点：

* 在一颗cgroup树里面，一个进程必须要属于一个cgroup。
* 新创建的子进程将会自动加入父进程所在的cgroup。
* 从一个cgroup移动一个进程到另一个cgroup时，只要有目的cgroup的写入权限就可以了，系统不会检查源cgroup里的权限。
* 用户只能操作属于自己的进程，不能操作其他用户的进程，root账号除外。

```bash
#--------------------------第一个shell窗口----------------------
#创建一个新的cgroup
dev@ubuntu:~/cgroup/demo$ sudo mkdir test
dev@ubuntu:~/cgroup/demo$ cd test

#将当前bash加入到上面新创建的cgroup中
dev@ubuntu:~/cgroup/demo/test$ echo $$
1421
dev@ubuntu:~/cgroup/demo/test$ sudo sh -c 'echo 1421 > cgroup.procs'
#注意：一次只能往这个文件中写一个进程ID，如果需要写多个的话，需要多次调用这个命令

#--------------------------第二个shell窗口----------------------
#重新打开一个shell窗口，避免第一个shell里面运行的命令影响输出结果
#这时可以看到cgroup.procs里面包含了上面的第一个shell进程
dev@ubuntu:~/cgroup/demo/test$ cat cgroup.procs
1421

#--------------------------第一个shell窗口----------------------
#回到第一个窗口，运行top命令
dev@ubuntu:~/cgroup/demo/test$ top
#这里省略输出内容

#--------------------------第二个shell窗口----------------------
#这时再在第二个窗口查看，发现top进程自动和它的父进程（1421）属于同一个cgroup
dev@ubuntu:~/cgroup/demo/test$ cat cgroup.procs
1421
16515
dev@ubuntu:~/cgroup/demo/test$ ps -ef|grep top
dev      16515  1421  0 04:02 pts/0    00:00:00 top
dev@ubuntu:~/cgroup/demo/test$

#在一颗cgroup树里面，一个进程必须要属于一个cgroup，
#所以我们不能凭空从一个cgroup里面删除一个进程，只能将一个进程从一个cgroup移到另一个cgroup，
#这里我们将1421移动到root cgroup
dev@ubuntu:~/cgroup/demo/test$ sudo sh -c 'echo 1421 > ../cgroup.procs'
dev@ubuntu:~/cgroup/demo/test$ cat cgroup.procs
16515
#移动1421到另一个cgroup之后，它的子进程不会随着移动

#--------------------------第一个shell窗口----------------------
##回到第一个shell窗口，进行清理工作
#先用ctrl+c退出top命令
dev@ubuntu:~/cgroup/demo/test$ cd ..
#然后删除创建的cgroup
dev@ubuntu:~/cgroup/demo$ sudo rmdir test
```
## 权限
上面我们都是用sudo(root账号)来操作的，但实际上普通账号也可以操作cgroup
```bash
#创建一个新的cgroup，并修改他的owner
dev@ubuntu:~/cgroup/demo$ sudo mkdir permission
dev@ubuntu:~/cgroup/demo$ sudo chown -R dev:dev ./permission/

#1421原来属于root cgroup，虽然dev没有root cgroup的权限，但还是可以将1421移动到新的cgroup下，
#说明在移动进程的时候，系统不会检查源cgroup里的权限。
dev@ubuntu:~/cgroup/demo$ echo 1421 > ./permission/cgroup.procs

#由于dev没有root cgroup的权限，再把1421移回root cgroup失败
dev@ubuntu:~/cgroup/demo$ echo 1421 > ./cgroup.procs
-bash: ./cgroup.procs: Permission denied

#找一个root账号的进程
dev@ubuntu:~/cgroup/demo$ ps -ef|grep /lib/systemd/systemd-logind
root       839     1  0 01:52 ?        00:00:00 /lib/systemd/systemd-logind
#因为该进程属于root，dev没有操作它的权限，所以将该进程加入到permission中失败
dev@ubuntu:~/cgroup/demo$ echo 839 >./permission/cgroup.procs
-bash: echo: write error: Permission denied
#只能由root账号添加
dev@ubuntu:~/cgroup/demo$ sudo sh -c 'echo 839 >./permission/cgroup.procs'

#dev还可以在permission下创建子cgroup
dev@ubuntu:~/cgroup/demo$ mkdir permission/c1
dev@ubuntu:~/cgroup/demo$ ls permission/c1
cgroup.clone_children  cgroup.procs  notify_on_release  tasks

#清理
dev@ubuntu:~/cgroup/demo$ sudo sh -c 'echo 839 >./cgroup.procs'
dev@ubuntu:~/cgroup/demo$ sudo sh -c 'echo 1421 >./cgroup.procs'
dev@ubuntu:~/cgroup/demo$ rmdir permission/c1
dev@ubuntu:~/cgroup/demo$ sudo rmdir permission
```

## cgroup.procs vs tasks
上面提到cgroup.procs包含的是进程ID， 而tasks里面包含的是线程ID，那么他们有什么区别呢？
```bash
#创建两个新的cgroup用于演示
dev@ubuntu:~/cgroup/demo$ sudo mkdir c1 c2

#为了便于操作，先给root账号设置一个密码，然后切换到root账号
dev@ubuntu:~/cgroup/demo$ sudo passwd root
dev@ubuntu:~/cgroup/demo$ su root
root@ubuntu:/home/dev/cgroup/demo#

#系统中找一个有多个线程的进程
root@ubuntu:/home/dev/cgroup/demo# ps -efL|grep /lib/systemd/systemd-timesyncd
systemd+   610     1   610  0    2 01:52 ?        00:00:00 /lib/systemd/systemd-timesyncd
systemd+   610     1   616  0    2 01:52 ?        00:00:00 /lib/systemd/systemd-timesyncd
#进程610有两个线程，分别是610和616

#将616加入c1/cgroup.procs
root@ubuntu:/home/dev/cgroup/demo# echo 616 > c1/cgroup.procs
#由于cgroup.procs存放的是进程ID，所以这里看到的是616所属的进程ID（610）
root@ubuntu:/home/dev/cgroup/demo# cat c1/cgroup.procs
610
#从tasks中的内容可以看出，虽然只往cgroup.procs中加了线程616，
#但系统已经将这个线程所属的进程的所有线程都加入到了tasks中，
#说明现在整个进程的所有线程已经处于c1中了
root@ubuntu:/home/dev/cgroup/demo# cat c1/tasks
610
616

#将616加入c2/tasks中
root@ubuntu:/home/dev/cgroup/demo# echo 616 > c2/tasks

#这时我们看到虽然在c1/cgroup.procs和c2/cgroup.procs里面都有610，
#但c1/tasks和c2/tasks中包含了不同的线程，说明这个进程的两个线程分别属于不同的cgroup
root@ubuntu:/home/dev/cgroup/demo# cat c1/cgroup.procs
610
root@ubuntu:/home/dev/cgroup/demo# cat c1/tasks
610
root@ubuntu:/home/dev/cgroup/demo# cat c2/cgroup.procs
610
root@ubuntu:/home/dev/cgroup/demo# cat c2/tasks
616
#通过tasks，我们可以实现线程级别的管理，但通常情况下不会这么用，
#并且在cgroup V2以后，将不再支持该功能，只能以进程为单位来配置cgroup

#清理
root@ubuntu:/home/dev/cgroup/demo# echo 610 > ./cgroup.procs
root@ubuntu:/home/dev/cgroup/demo# rmdir c1
root@ubuntu:/home/dev/cgroup/demo# rmdir c2
root@ubuntu:/home/dev/cgroup/demo# exit
exit
```

## release_agent
当一个cgroup里没有进程也没有子cgroup时，release_agent将被调用来执行cgroup的清理工作。

```bash
#创建新的cgroup用于演示
dev@ubuntu:~/cgroup/demo$ sudo mkdir test
#先enable release_agent
dev@ubuntu:~/cgroup/demo$ sudo sh -c 'echo 1 > ./test/notify_on_release'

#然后创建一个脚本/home/dev/cgroup/release_demo.sh，
#一般情况下都会利用这个脚本执行一些cgroup的清理工作，但我们这里为了演示简单，仅仅只写了一条日志到指定文件
dev@ubuntu:~/cgroup/demo$ cat > /home/dev/cgroup/release_demo.sh << EOF
#!/bin/bash
echo \$0:\$1 >> /home/dev/release_demo.log
EOF

#添加可执行权限
dev@ubuntu:~/cgroup/demo$ chmod +x ../release_demo.sh

#将该脚本设置进文件release_agent
dev@ubuntu:~/cgroup/demo$ sudo sh -c 'echo /home/dev/cgroup/release_demo.sh > ./release_agent'
dev@ubuntu:~/cgroup/demo$ cat release_agent
/home/dev/cgroup/release_demo.sh

#往test里面添加一个进程，然后再移除，这样就会触发release_demo.sh
dev@ubuntu:~/cgroup/demo$ echo $$
27597
dev@ubuntu:~/cgroup/demo$ sudo sh -c 'echo 27597 > ./test/cgroup.procs'
dev@ubuntu:~/cgroup/demo$ sudo sh -c 'echo 27597 > ./cgroup.procs'

#从日志可以看出，release_agent被触发了，/test是cgroup的相对路径
dev@ubuntu:~/cgroup/demo$ cat /home/dev/release_demo.log
/home/dev/cgroup/release_demo.sh:/test
```

## 结束语
本文介绍了如何操作cgroup，由于没有和任何subsystem关联，所以在这颗树上的所有操作都没有实际的功能，不会对系统有影响。从下一篇开始，将介绍具体的subsystem。

## 参考
* [CGROUPS v1](https://www.kernel.org/doc/Documentation/cgroup-v1/cgroups.txt)
