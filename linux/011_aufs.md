#Linux文件系统之aufs
aufs的全称是advanced multi-layered unification filesystem，主要功能是把多个文件夹的内容合并到一起，提供一个统一的视图，主要用于各个Linux发行版的livecd中，以及docker里面用来组织image。

据说由于aufs代码的可维护性不好（代码可读性和注释不太好），所以一直没有被合并到Linux内核的主线中去，不过有些发行版的kernel里面维护的有该文件系统，比如在ubuntu 16.04的内核代码中，就有该文件系统。

>本篇所有例子都在ubuntu-server-x86_64 16.04下执行通过

##检查系统是否支持aufs
在使用aufs之前，可以通过下面的命令确认当前系统是否支持aufs，如果不支持，请自行根据相应发行版的文档安装
```bash
#下面的命令如果没有输出，表示内核不支持aufs
#由于ubuntu 16.04的内核中已经将aufs编译进去了，所以默认就支持
dev@ubuntu:~$ grep aufs /proc/filesystems
nodev   aufs
#这里nodev表示该文件系统不需要建在设备上
```

>注意：有些Linux发行版可能将aufs编译成了模块，所以虽然这里显示内核不支持，但其实后面的命令都能正常运行

##挂载aufs
选择好相应的参数（参考[帮助文档](http://manpages.ubuntu.com/manpages/xenial/en/man5/aufs.5.html)），调用mount命令即可，示例如下
```
# mount -t aufs -o br=./Branch-0:./Branch-0:./Branch-2 none ./MountPoint
```

* -t aufs： 指定挂载类型为aufs
* -o br=./Branch-0:./Branch-0:./Branch-2： 表示将当前目录下的Branch-0，Branch-1，Branch-2三个文件夹联合到一起
* none：aufs不需要设备，只依赖于-o br指定的文件夹，所以这里填none即可
* ./MountPoint：表示将最后联合的结果挂载到当前的MountPoint目录下，然后我们就可以往这个目录里面读写文件了

假设Branch-0里面有文件001.txt、003.txt，Branch-1里面有文件001.txt、003.txt、004.txt，Branch-2里面有文件002.txt、003.txt。

mount完成后，得到的结果将会如下图所示
```    
              /*001.txt(b0)表示Branch-0的001.txt文件，其它的以此类推*/
           +-------------+-------------+-------------+-------------+
MountPoint | 001.txt(b0) | 002.txt(b2) | 003.txt(b0) | 004.txt(b1) |    
           +-------------+-------------+-------------+-------------+
                  ↑             ↑             ↑             ↑
                  |             |             |             |
           +-------------+-------------+-------------+-------------+
Branch-0   |   001.txt   |             |   003.txt   |             |
           +-------------+-------------+-------------+-------------+
Branch-1   |   001.txt   |             |   003.txt   |   004.txt   |
           +-------------+-------------+-------------+-------------+
Branch-2   |             |   002.txt   |   003.txt   |             |
           +-------------+-------------+-------------+-------------+
```

联合之后，在MountPoint下将会看到四个文件，分别是Branch-0下的001.txt、003.txt，Branch-1下的04.txt，以及Branch-2下的002.txt。

* branch是aufs里面的概念，其实一个branch就是一个目录，所以上面的Branch-0,1,2就是三个目录
* branch是有index的，index越小的branch会放在最上面，如果多个branch里面有同样的文件，只有index最小的那个branch下的文件才会被访问到
* MountPoint就是最后这三个目录联合后挂载到的位置，访问这个目录下的文件都会经过aufs文件系统，换句话说，直接访问Branch-0,1,2这三个目录的话，aufs是不知道的

>注意：并不是所有文件系统里的目录都能作为aufs的branch，目前aufs不支持的有：btrfs aufs eCryptfs


读这些文件的时候访问的是最上层的文件，但如果要写这些文件呢？或者在挂载点下创建新的文件呢？请看下面的示例

##只读挂载
挂载时，可以指定每个branch的读写权限，如果不指定的话，第一个目录将会是可写的，其它的目录是只读的，在实际使用时，最好是显示的指定每个branch的读写属性，这样大家都一眼就能看懂。这里先演示一下只读挂载：

```bash
#准备相应的目录和文件
dev@ubuntu:~$ mkdir /tmp/aufs && cd /tmp/aufs
dev@ubuntu:/tmp/aufs$ mkdir dir0 dir1 root
dev@ubuntu:/tmp/aufs$ echo dir0 > dir0/001.txt
dev@ubuntu:/tmp/aufs$ echo dir0 > dir0/002.txt
dev@ubuntu:/tmp/aufs$ echo dir1 > dir1/002.txt
dev@ubuntu:/tmp/aufs$ echo dir1 > dir1/003.txt
#最后用tree命令来看看最终的目录结构
dev@ubuntu:/tmp/aufs$ tree
.
├── dir0
│   ├── 001.txt
│   └── 002.txt
├── dir1
│   ├── 002.txt
│   └── 003.txt
└── root

#通过指定ro参数来让两个branch都为只读
dev@ubuntu:/tmp/aufs$ sudo mount -t aufs -o br=./dir0=ro:./dir1=ro none ./root
#联合后最终的root目录下将看到三个文件
dev@ubuntu:/tmp/aufs$ ls root/
001.txt  002.txt  003.txt
#其中002.txt的内容是dir0中的002.txt的内容，说明dir0的index要比dir1的index小
dev@ubuntu:/tmp/aufs$ cat root/002.txt
dir0

#由于是只读挂载，所以touch失败
dev@ubuntu:/tmp/aufs$ touch root/001.txt
touch: cannot touch 'root/001.txt': Read-only file system
dev@ubuntu:/tmp/aufs$ touch root/003.txt
touch: cannot touch 'root/003.txt': Read-only file system

#但是我们可以跳过root目录来修改001.txt和003.txt，
#因为跳过了root目录，所以就不受aufs控制
dev@ubuntu:/tmp/aufs$ touch dir0/001.txt
dev@ubuntu:/tmp/aufs$ touch dir1/003.txt

#我们还能在下面的目录中创建新的文件
dev@ubuntu:/tmp/aufs$ touch dir1/004.txt
#新创建的文件能及时的反应到挂载点上去
dev@ubuntu:/tmp/aufs$ ls ./root/
001.txt  002.txt  003.txt  004.txt

#删除该文件，以免影响后续的演示
dev@ubuntu:/tmp/aufs$ rm ./dir1/004.txt
```

从上面的演示可以看出，我们可以跳过挂载点直接读写底层的目录，这样就不受aufs的控制，但我们修改的内容（dir1里面创建的004.txt）还是能在挂载点下看到，这是因为aufs在访问文件时，默认的做法是如果最上层目录里面没这个文件，就一层一层的往下找，所以下层有变动的话，aufs会自动发现。控制这种行为的参数为“udba”，有兴趣可以参考[帮助文档](http://manpages.ubuntu.com/manpages/xenial/en/man5/aufs.5.html)

>由于访问一个文件时需要一级一级往下找，所以如果联合的目录（层级）过多的话，会影响性能

##读写挂载
如果联合的文件夹有写的权限，那么所有的修改都会写入可写的那个文件夹，如果可写的文件夹有多个，那么写入哪个文件夹就依赖于相应的策略，有round-robin、最多剩余空间等，详情请参考[帮助文档](http://manpages.ubuntu.com/manpages/xenial/en/man5/aufs.5.html)中的“create”参数，这里不做介绍。

```bash
dev@ubuntu:/tmp/aufs$ sudo umount ./root
#dir0具有读写权限，dir1为只读权限
dev@ubuntu:/tmp/aufs$ sudo mount -t aufs -o br=./dir0=rw:./dir1=ro none ./root
dev@ubuntu:/tmp/aufs$ echo "root->write" >> ./root/001.txt
dev@ubuntu:/tmp/aufs$ echo "root->write" >> ./root/002.txt
dev@ubuntu:/tmp/aufs$ echo "root->write" >> ./root/003.txt
dev@ubuntu:/tmp/aufs$ echo "root->write" >> ./root/005.txt

#跟开始前相比，dir0目录下多了003.txt和005.txt，其它的保持不变
dev@ubuntu:/tmp/aufs$ ls ./root/
001.txt  002.txt  003.txt  005.txt
dev@ubuntu:/tmp/aufs$ ls ./dir0/
001.txt  002.txt  003.txt  005.txt
dev@ubuntu:/tmp/aufs$ ls ./dir1/
002.txt  003.txt

#再来看看内容，dir1里面的内容保持不变
dev@ubuntu:/tmp/aufs$ cat ./dir1/002.txt
dir1
dev@ubuntu:/tmp/aufs$ cat ./dir1/003.txt
dir1

#dir0下的文件内容都变了
dev@ubuntu:/tmp/aufs$ cat ./dir0/001.txt
dir0
root->write
dev@ubuntu:/tmp/aufs$ cat ./dir0/002.txt
dir0
root->write
dev@ubuntu:/tmp/aufs$ cat ./dir0/003.txt
dir1
root->write
dev@ubuntu:/tmp/aufs$ cat ./dir0/005.txt
root->write
```

* 当创建一个新文件的时候，新的文件会写入具有rw权限的那个目录，如果有多个目录具有rw权限，那么依赖于挂载时配置的的创建策略
* 当修改一个具有rw权限目录下的文件时，直接修改该文件
* 当修改一个只有ro权限目录下的文件时，aufs会先将该文件拷贝到一个rw权限的目录里面，然后在上面进行修改，这就是所谓的COW(copy on write)，拷贝的速度依赖于底层branch所在的文件系统。

从上面可以看出，COW对于大文件来说，性能还是很低的，同时也会占用很多的空间，但由于只需要在第一次修改的时候拷贝一次，所以很多情况下还是能接受。

##删除文件
删除文件时，如果该文件只在rw目录下有，那就直接删除rw目录下的该文件，如果该文件在ro目录下有，那么aufs将会在rw目录里面创建一个.wh开头的文件，标识该文件已被删除

```bash
#通过aufs删除所有文件
dev@ubuntu:/tmp/aufs$ rm ./root/001.txt ./root/002.txt ./root/003.txt ./root/005.txt

#dir0下的文件全被删除了，但dir1目录下的文件没动
dev@ubuntu:/tmp/aufs$ tree
.
├── dir0
├── dir1
│   ├── 002.txt
│   └── 003.txt
└── root

#通过-a参数来看看dir0目录下的内容
#可以看到aufs为002.txt和003.txt新建了两个特殊的以.wh开头的文件，
#用来表示这两个文件已经被删掉了
#这里其他.wh开头的文件都是aufs用到的一些属性文件
dev@ubuntu:/tmp/aufs$ ls ./dir0/ -a
.  ..  .wh.002.txt  .wh.003.txt  .wh..wh.aufs  .wh..wh.orph  .wh..wh.plnk
```

##结束语
这里只介绍了aufs的基本功能，其它的高级配置项没有涉及，比如动态的增加和删除branch等。

使用aufs时，建议参考livecd及docker的使用方式，就是将所有的目录都以只读的方式和一个支持读写的空目录联合起来，这样所有的修改都会存到那个指定的空目录中，不用之后删除掉那个目录就可以了，并且在使用的过程中不要绕过aufs直接操作底层的branch，也不要动态的增加和删除branch，如果把使用场景弄得太复杂，由于aufs里面的细节很多，很有可能会由于对aufs的理解不深而踩坑。

##参考
[aufs](http://aufs.sourceforge.net/)
[aufs manual](http://manpages.ubuntu.com/manpages/xenial/en/man5/aufs.5.html)
[Linux AuFS Examples](http://www.thegeekstuff.com/2013/05/linux-aufs/)