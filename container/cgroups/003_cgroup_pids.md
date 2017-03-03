# Linux Cgroup系列（03）：限制cgroup的进程数（subsystem之pids）

[上一篇文章](https://segmentfault.com/a/1190000007241437)中介绍了如何管理cgroup，从这篇开始将介绍具体的subsystem。

本篇将介绍一个简单的subsystem，名字叫[pids](https://www.kernel.org/doc/Documentation/cgroup-v1/pids.txt)，功能是限制cgroup及其所有子孙cgroup里面能创建的总的进程数量。

>本篇所有例子都在ubuntu-server-x86_64 16.04下执行通过

##创建子cgroup
在ubuntu 16.04里面，systemd已经帮我们将各个subsystem和cgroup树绑定并挂载好了，我们直接用现成的就可以了。
```bash
#从这里的输出可以看到，pids已经被挂载在了/sys/fs/cgroup/pids，这是systemd做的
dev@dev:~$ mount|grep pids
cgroup on /sys/fs/cgroup/pids type cgroup (rw,nosuid,nodev,noexec,relatime,pids)
```

创建子cgroup，取名为test
```bash
#进入目录/sys/fs/cgroup/pids/并新建一个目录，即创建了一个子cgroup
dev@dev:~$ cd /sys/fs/cgroup/pids/
dev@dev:/sys/fs/cgroup/pids$ sudo mkdir test
#这里将test目录的owner设置成dev账号，这样后续操作就不用每次都敲sudo了，省去麻烦
dev@dev:/sys/fs/cgroup/pids$ sudo chown -R dev:dev ./test/
```

再来看看test目录下的文件
```bash
#除了上一篇中介绍的那些文件外，多了两个文件
dev@dev:/sys/fs/cgroup/pids$ cd test
dev@dev:/sys/fs/cgroup/pids/test$ ls
cgroup.clone_children  cgroup.procs  notify_on_release  pids.current  pids.max  tasks
```

下面是这两个文件的含义：

* pids.current: 表示当前cgroup及其所有子孙cgroup中现有的总的进程数量
    ```bash
    #由于这是个新创建的cgroup，所以里面还没有任何进程
    dev@dev:/sys/fs/cgroup/pids/test$ cat pids.current 
    0
    ```

* pids.max: 当前cgroup及其所有子孙cgroup中所允许创建的总的最大进程数量，在根cgroup下没有这个文件，原因显而易见，因为我们没有必要限制整个系统所能创建的进程数量。
    ```bash
    #max表示没做任何限制
    dev@dev:/sys/fs/cgroup/pids/test$ cat pids.max 
    max
    ```

##限制进程数
这里我们演示一下如何让限制功能生效
```bash
#--------------------------第一个shell窗口----------------------
#将pids.max设置为1，即当前cgroup只允许有一个进程
dev@dev:/sys/fs/cgroup/pids/test$ echo 1 > pids.max
#将当前bash进程加入到该cgroup
dev@dev:/sys/fs/cgroup/pids/test$ echo $$ > cgroup.procs
#--------------------------第二个shell窗口----------------------
#重新打开一个bash窗口，在里面看看cgroup “test”里面的一些数据
#因为这是一个新开的bash，跟cgroup ”test“没有任何关系，所以在这里运行命令不会影响cgroup “test”
dev@dev:~$ cd /sys/fs/cgroup/pids/test
#设置的最大进程数是1
dev@dev:/sys/fs/cgroup/pids/test$ cat pids.max
1
#目前test里面已经有了一个进程，说明不能在fork或者clone进程了
dev@dev:/sys/fs/cgroup/pids/test$ cat pids.current
1
#这个进程就是第一个窗口的bash
dev@dev:/sys/fs/cgroup/pids/test$ cat cgroup.procs
3083
#--------------------------第一个shell窗口----------------------
#回到第一个窗口，随便运行一个命令，由于当前pids.current已经等于pids.max了，
#所以创建新进程失败，于是命令运行失败，说明限制生效
dev@dev:/sys/fs/cgroup/pids/test$ ls
-bash: fork: retry: No child processes
-bash: fork: retry: No child processes
-bash: fork: retry: No child processes
-bash: fork: retry: No child processes
-bash: fork: Resource temporarily unavailable
```

##当前cgroup和子cgroup之间的关系
当前cgroup中的pids.current和pids.max代表了当前cgroup及所有子孙cgroup的所有进程，所以子孙cgroup中的pids.max大小不能超过父cgroup中的大小，如果子cgroup中的pids.max设置的大于父cgroup里的大小，会怎么样？请看下面的演示
```bash
#继续使用上面的两个窗口
#--------------------------第二个shell窗口----------------------
#将pids.max设置成2
dev@dev:/sys/fs/cgroup/pids/test$ echo 2 > pids.max
#在test下面创建一个子cgroup
dev@dev:/sys/fs/cgroup/pids/test$ mkdir subtest
dev@dev:/sys/fs/cgroup/pids/test$ cd subtest/
#将subtest的pids.max设置为5
dev@dev:/sys/fs/cgroup/pids/test/subtest$ echo 5 > pids.max
#将当前bash进程加入到subtest中
dev@dev:/sys/fs/cgroup/pids/test/subtest$ echo $$ > cgroup.procs
#--------------------------第三个shell窗口----------------------
#重新打开一个bash窗口，看一下test和subtest里面的数据
#test里面的数据如下：
dev@dev:~$ cd /sys/fs/cgroup/pids/test
dev@dev:/sys/fs/cgroup/pids/test$ cat pids.max
2
#这里为2表示目前test和subtest里面总的进程数为2
dev@dev:/sys/fs/cgroup/pids/test$ cat pids.current
2
dev@dev:/sys/fs/cgroup/pids/test$ cat cgroup.procs
3083

#subtest里面的数据如下：
dev@dev:/sys/fs/cgroup/pids/test$ cat subtest/pids.max
5
dev@dev:/sys/fs/cgroup/pids/test$ cat subtest/pids.current
1
dev@dev:/sys/fs/cgroup/pids/test$ cat subtest/cgroup.procs
3185
#--------------------------第一个shell窗口----------------------
#回到第一个窗口，随便运行一个命令，由于test里面的pids.current已经等于pids.max了，
#所以创建新进程失败，于是命令运行失败，说明限制生效
dev@dev:/sys/fs/cgroup/pids/test$ ls
-bash: fork: retry: No child processes
-bash: fork: retry: No child processes
-bash: fork: retry: No child processes
-bash: fork: retry: No child processes
-bash: fork: Resource temporarily unavailable
#--------------------------第二个shell窗口----------------------
#回到第二个窗口，随便运行一个命令，虽然subtest里面的pids.max还大于pids.current，
#但由于其父cgroup “test”里面的pids.current已经等于pids.max了，
#所以创建新进程失败，于是命令运行失败，说明子cgroup中的进程数不仅受自己的pids.max的限制，
#还受祖先cgroup的限制
dev@dev:/sys/fs/cgroup/pids/test/subtest$ ls
-bash: fork: retry: No child processes
-bash: fork: retry: No child processes
-bash: fork: retry: No child processes
-bash: fork: retry: No child processes
-bash: fork: Resource temporarily unavailable
```
##pids.current > pids.max的情况
并不是所有情况下都是pids.max >= pids.current，在下面两种情况下，会出现pids.max < pids.current 的情况：

* 设置pids.max时，将其值设置的比pids.current小
    ```bash
    #继续使用上面的三个窗口
    #--------------------------第三个shell窗口----------------------
    #将test的pids.max设置为1
    dev@dev:/sys/fs/cgroup/pids/test$ echo 1 > pids.max
    dev@dev:/sys/fs/cgroup/pids/test$ cat pids.max
    1
    #这个时候就会出现pids.current > pids.max的情况
    dev@dev:/sys/fs/cgroup/pids/test$ cat pids.current
    2

    #--------------------------第一个shell窗口----------------------
    #回到第一个shell
    #还是运行失败，说明虽然pids.current > pids.max，但限制创建新进程的功能还是会生效
    dev@dev:/sys/fs/cgroup/pids/test$ ls
    -bash: fork: retry: No child processes
    -bash: fork: retry: No child processes
    -bash: fork: retry: No child processes
    -bash: fork: retry: No child processes
    -bash: fork: Resource temporarily unavailable
    ```
* pids.max只会在当前cgroup中的进程fork、clone的时候生效，将其他进程加入到当前cgroup时，不会检测pids.max，所以将其他进程加入到当前cgroup有可能会导致pids.current > pids.max
    ```bash
    #继续使用上面的三个窗口
    #--------------------------第三个shell窗口----------------------
    #将subtest中的进程移动到根cgroup下，然后删除subtest
    dev@dev:/sys/fs/cgroup/pids/test$ sudo sh -c 'echo 3185 > /sys/fs/cgroup/pids/cgroup.procs'
    #里面没有进程了，说明移动成功
    dev@dev:/sys/fs/cgroup/pids/test$ cat subtest/cgroup.procs
    #移除成功
    dev@dev:/sys/fs/cgroup/pids/test$ rmdir subtest/

    #这时候test下的pids.max等于pids.current了
    dev@dev:/sys/fs/cgroup/pids/test$ cat pids.max
    1
    dev@dev:/sys/fs/cgroup/pids/test$ cat pids.current
    1

    #--------------------------第二个shell窗口----------------------
    #将当前bash加入到test中
    dev@dev:/sys/fs/cgroup/pids/test/subtest$ cd ..
    dev@dev:/sys/fs/cgroup/pids/test$ echo $$ > cgroup.procs

    #--------------------------第三个shell窗口----------------------
    #回到第三个窗口，查看相关信息
    #第一个和第二个窗口的bash都属于test
    dev@dev:/sys/fs/cgroup/pids/test$ cat cgroup.procs
    3083
    3185
    dev@dev:/sys/fs/cgroup/pids/test$ cat pids.max
    1
    #出现了pids.current > pids.max的情况，这是因为我们将第二个窗口的shell加入了test
    dev@dev:/sys/fs/cgroup/pids/test$ cat pids.current
    2
    #--------------------------第二个shell窗口----------------------
    #对fork调用的限制仍然生效
    dev@dev:/sys/fs/cgroup/pids/test$ ls
    -bash: fork: retry: No child processes
    -bash: fork: retry: No child processes
    -bash: fork: retry: No child processes
    -bash: fork: retry: No child processes
    -bash: fork: Resource temporarily unavailable
    ```

清理
```bash
#--------------------------第三个shell窗口----------------------
dev@dev:/sys/fs/cgroup/pids/test$ sudo sh -c 'echo 3185 > /sys/fs/cgroup/pids/cgroup.procs'
dev@dev:/sys/fs/cgroup/pids/test$ sudo sh -c 'echo 3083 > /sys/fs/cgroup/pids/cgroup.procs'
dev@dev:/sys/fs/cgroup/pids/test$ cd ..
dev@dev:/sys/fs/cgroup/pids$ sudo rmdir test/
```

##结束语
本文介绍了如何利用pids这个subsystem来限制cgroup中的进程数，以及一些要注意的地方，总的来说pids比较简单。下一篇将介绍稍微复杂点的内存控制。

##参考
[Process Number Controller](https://www.kernel.org/doc/Documentation/cgroup-v1/pids.txt)
