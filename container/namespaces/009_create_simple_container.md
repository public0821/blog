#Linux Namespace系列（09）：利用Namespace创建一个简单可用的容器

本文将演示如何利用namespace创建一个完整的容器，并在里面运行busybox。如果对namespace不是很熟悉，请先参考前面几遍介绍不同类型namespace的文章。

busybox是一个Linux下的可执行程序，采用静态链接，不依赖于其他任何动态库。他里面实现了一些Linux下常用的命令，如ls，hostname，date，ps，mount等等，详细的介绍请参考它的[官网](https://busybox.net/)。

>本篇所有例子都在ubuntu-server-x86_64 16.04下执行通过

##准备container的根目录
```bash
#创建一个单独的目录，后续所有的操作都在该目录下进行，目录名称无特殊要求
dev@ubuntu:~$ mkdir chroot && cd chroot
#下载编译好的busybox可执行文件
dev@ubuntu:~/chroot$ wget https://busybox.net/downloads/binaries/1.21.1/busybox-x86_64
#创建new_root/bin目录，new_root将会是新容器的根目录，bin目录用来放busybox
#由于/bin默认就在PATH中，所以里面放的程序都可以直接在shell里面执行，不需要带完整的路径
dev@ubuntu:~/chroot$ mkdir -p new_root/bin
dev@ubuntu:~/chroot$ chmod +x ./busybox-x86_64

#将busybox-x86_64移到bin目录下，并重命名为busybox
dev@ubuntu:~/chroot$ mv busybox-x86_64 new_root/bin/busybox
#运行ls试试，确保busybox能正常工作
dev@ubuntu:~/chroot$ ./new_root/bin/busybox ls
new_root

#安装busybox到bin目录，不安装的话每次执行ls命令都需要使用上面那种格式： busybox ls
#安装之后就会创建一个ls到busybox的硬链接，这样执行ls的时候就不用再输入前面的busybox了
dev@ubuntu:~/chroot$ ./new_root/bin/busybox --install ./new_root/bin/

#运行下bin下面的ls，确保安装成功
dev@ubuntu:~/chroot$ ls -l ./new_root/bin/ls
-rwxrwxr-x 348 dev dev 973200 7月   9  2013 ./new_root/bin/ls
dev@ubuntu:~/chroot$ ./new_root/bin/ls
new_root

#使用chroot命令，切换根目录
dev@ubuntu:~/chroot$ sudo chroot ./new_root/ sh

#切换成功，由于new_root下面只有busybox，没有任何配置文件，
#所以shell的提示符里只包含当前目录
#尝试运行几个命令，一切正常
/ # ls
bin
/ # which ls
/bin/ls
/ # hostname
ubuntu
/ # id
uid=0 gid=0 groups=0
/ # exit
dev@ubuntu:~/chroot$ 
```

##创建容器并做相关配置
```bash
#新建/data目录用来在主机和容器之间共享数据
dev@ubuntu:~/chroot$ sudo mkdir /data
dev@ubuntu:~/chroot$ sudo chown dev:dev /data
dev@ubuntu:~/chroot$ touch /data/001

#创建新的容器，指定所有namespace相关的参数，
#这里--propagation private是为了让容器里的mount point都变成private的，
#这是因为pivot_root命令需要原来根目录的挂载点为private，
#只有我们需要在host和container之间共享挂载信息的时候，才需要使用shared或者slave类型
dev@ubuntu:~/chroot$ unshare --user --mount --ipc --pid --net --uts -r --fork --propagation private bash
#设置容器的主机名
root@ubuntu:~/chroot# hostname container01
root@ubuntu:~/chroot# exec bash

#创建old_root用于pivot_root命令，创建data目录用于绑定/data目录
root@container01:~/chroot# mkdir -p ./new_root/old_root/ ./new_root/data/

#由于pivot_root命令要求老的根目录和新的根目录不能在同一个挂载点下，
#所以这里利用bind mount，在原地创建一个新的挂载点
root@container01:~/chroot# mount --bind ./new_root/ ./new_root/

#将/data目录绑定到new_root/data，这样pivot_root后，就能访问/data下的东西了
root@container01:~/chroot# mount --bind /data ./new_root/data

#进入new_root目录，然后切换根目录
root@container01:~/chroot# cd new_root/
root@container01:~/chroot/new_root# pivot_root ./ ./old_root/

#但shell提示符里显示的当前目录还是原来的目录，没有切换到‘/’下，
#这是因为当前运行的shell还是host里面的bash
root@container01:~/chroot/new_root# ls
bin  data  old_root

#重新加载new_root下面的shell，这样contianer和host就没有关系了，
#从shell提示符中可以看出，当前目录已经变成了‘/’
root@container01:~/chroot/new_root# exec sh
/ # 
#由于没有/etc目录，也就没有相关的profile，于是shell的提示符里面只包含当前路径。

#设置PS1环境变量，让shell提示符好看点，这里直接写了root在提示符里面，
#是因为我们新的container里面没有账号相关的配置文件，
#虽然系统知道当前账号的ID是0，但不知道账号的用户名是什么。
/ # export PS1='root@$(hostname):$(pwd)# '
root@container01:/# 

#没有/etc目录，没有user相关的配置文件，所以不知道ID为0的用户名是什么
root@container01:/# whoami
whoami: unknown uid 0

#mount命令依赖于/proc目录，所以这里mount操作失败
root@container01:/# mount
mount: no /proc/mounts

#重新mount /proc
root@container01:/# mkdir /proc
root@container01:/# mount -t proc none /proc

#这时可以看到所有的mount信息了，从host复制过来的mount信息都挂载在/old_root目录下
root@container01:/# mount
/dev/mapper/ubuntu--vg-root on /old_root type ext4 (rw,relatime,errors=remount-ro,data=ordered)
udev on /old_root/dev type devtmpfs (rw,nosuid,relatime,size=1005080k,nr_inodes=251270,mode=755)
devpts on /old_root/dev/pts type devpts (rw,nosuid,noexec,relatime,mode=600,ptmxmode=000)
tmpfs on /old_root/dev/shm type tmpfs (rw,nosuid,nodev)
......
/dev/mapper/ubuntu--vg-root on / type ext4 (rw,relatime,errors=remount-ro,data=ordered)
/dev/mapper/ubuntu--vg-root on /data type ext4 (rw,relatime,errors=remount-ro,data=ordered)
none on /proc type proc (rw,nodev,relatime)

#umount掉/old_root下的所有mount point
root@container01:/# umount -l /old_root
#这时候就只剩下根目录，/proc，/data三个挂载点了
root@container01:/# mount
/dev/mapper/ubuntu--vg-root on / type ext4 (rw,relatime,errors=remount-ro,data=ordered)
/dev/mapper/ubuntu--vg-root on /data type ext4 (rw,relatime,errors=remount-ro,data=ordered)
none on /proc type proc (rw,nodev,relatime)

#试试cd命令，提示失败， $HOME还是指向老的/home/dev，
#除了$HOME之外，还有其他一些环境变量也有同样的问题。
#这主要是由于我们新的container中缺少配置文件，导致很多环境变量没有更新。
root@container01:/# cd
sh: cd: can not cd to /home/dev

#试试ps，显示的是container里面启动的进程
root@container01:/# ps
PID   USER     TIME   COMMAND
    1 0          0:00 sh
   55 0          0:00 ps

#touch文件001成功
root@container01:/# touch /data/001
#新创建一个002文件
root@container01:/# touch /data/002
root@container01:/# ls /data
001  002

#退出contianer01，在/data目录能看到我们上面在container01种创建的002文件
root@container01:/# exit
dev@ubuntu:~/chroot$ ls /data
001  002
```

##总结

本文利用busybox和pivot_root演示了如何创建一个简单的容器，并且实现了在host和container之间共享文件夹。这个容器的功能非常简单，很多目录都没有构建，导致只能运行busybox里面的部分命令，有些命令运行时会有异常。要想构造一个完整易用的容器，还需要很多工作要做，这里只演示了冰山一角，在后续的“docker系列”中，将深入分析docker是如何一步一步帮助我们构建安全易用的contianer的，敬请期待。

##参考
[Video : Cgroups, namespaces, and beyond: what are containers made from?](https://www.youtube.com/watch?v=sK5i-N34im8)
