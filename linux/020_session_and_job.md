# Linux session和进程组概述

在[上一篇](https://segmentfault.com/a/1190000009082089)中介绍了tty的相关原理，这篇将介绍跟tty密切相关的session和进程组。

>本篇主要目的是澄清一些概念，不涉及细节

## session
session就是一组进程的集合，session id就是这个session中leader的进程ID。

### session的特点
session的主要特点是当session的leader退出后，session中的所有其它进程将会收到SIGHUP信号，其默认行为是终止进程，即session的leader退出后，session中的其它进程也会退出。

如果session和tty关联的话，它们之间只能一一对应，一个tty只能属于一个session，一个session只能打开一个tty。当然session也可以不和任何tty关联。

### session的创建
session可以在任何时候创建，调用setsid函数即可，session中的第一个进程即为这个session的leader，leader是不能变的。常见的创建session的场景是：

* 用户登录后，启动shell时将会创建新的session，shell会作为session的leader，随后shell里面运行的进程都将属于这个session，当shell退出后，所有该用户运行的进程将退出。这类session一般都会和一个特定的tty关联，session的leader会成为tty的控制进程，当session的前端进程组发生变化时，控制进程负责更新tty上关联的前端进程组，当tty要关闭的时候，控制进程所在session的所有进程都会收到SIGHUP信号。
* 启动deamon进程，这类进程需要和父进程划清界限，所以需要启动一个新的session。这类session一般不会和任何tty关联。

## 进程组
进程组（process group）也是一组进程的集合，进程组id就是这个进程组中leader的进程ID。

### 进程组的特点
进程组的主要特点是可以以进程组为单位通过函数[killpg](http://man7.org/linux/man-pages/man3/killpg.3.html)发送信号

### 进程组的创建
进程组主要用在shell里面，shell负责进程组的管理，包括创建、销毁等。（这里shell就是session的leader）

* 对大部分进程来说，它自己就是进程组的leader，并且进程组里面就只有它自己一个进程
* shell里面执行类似```ls|more```这样的以管道连接起来的命令时，两个进程就属于同一个进程组，ls是进程组的leader。
* shell里面启动一个进程后，一般都会将该进程放到一个单独的进程组，然后该进程fork的所有进程都会属于该进程组，比如多进程的程序，它的所有进程都会属于同一个进程组，当在shell里面按下CTRL+C时，该程序的所有进程都会收到SIGINT而退出。

### 后台进程组
shell中启动一个进程时，默认情况下，该进程就是一个前端进程组的leader，可以收到用户的输入，并且可以将输出打印到终端，只有当该进程组退出后，shell才可以再响应用户的输入。

但我们也可以将该进程组运行在后台，这样shell就可以继续相应用户的输入，常见的方法如下：

* 启动程序时，在后面加&，如```sleep 1000 &```，进程将会进入后台继续运行
* 程序启动后，可以按CTRL+Z让它进入后台，和后面加&不同的是，进程会被暂停执行

对于后台运行的进程组，在shell里面体现为job的概念，即一个后台进程组就是一个job，job有如下限制：

* 默认情况下，只要后台进程组的任何一个进程读tty，将会使整个进程组的所有进程暂停
* 默认情况下，只要后台进程组的任何一个进程写tty，将有可能会使整个进程组的所有进程暂停（依赖于tty的配置，请参考[TTY/PTS概述](https://segmentfault.com/a/1190000009082089)）

所有后台运行的进程组可以通过jobs命令查看到，也可以通过fg命令将后台进程组切换到前端，这样就可以继续接收用户的输入了。这两个命令的具体用法请参考它们的帮助文件，这里只给出一个简单的例子：
```bash
#通常情况下，sleep命令会一直等待在那里，直到指定的时间过去后才退出。
#shell启动sleep程序时，就将sleep放到了一个新的进程组，
#并且该进程组为前端进程组，虽然sleep不需要输入，也没有输出，
#但当前session的标准输入和输出还是归它，别人用不了，
#只有我们按下CTRL+C使sleep进程退出后，shell自己重新变成了前端进程组，
#于是shell重新具备了响应输入以及输出能力
dev@debian:~$ sleep 1000
^C

#我们可以在命令行的后面加上&符号，shell还是照样会创建新的进程组，
#并且sleep进程就是新进程组的leader，
#但是shell会将sleep进程组放到后端，让它成为后台进程组
#这里[1]是job id，1627是进程组的ID，即sleep进程的id
dev@debian:~$ sleep 1000 &
[1] 1627

#可以通过jobs命令看到当前有哪些后台进程组（job）
dev@debian:~$ jobs
[1]+  Running                 sleep 1000 &

#使用fg命令带上job id，即可让后端进程组回到前端，
#然后我们使用CTRL+Z命令可以让它再次回到后端，并暂停进程的执行
#CTRL+Z和&不一样的地方就是CTRL+Z会让进程暂停执行，而&不会
dev@debian:~$ fg 1
sleep 1000
^Z
[1]+  Stopped                 sleep 1000
#Stopped状态表示进程在后台已经暂停执行了
dev@debian:~$ jobs
[1]+  Stopped                 sleep 1000

``` 

## session和进程组的关系
deamon程序虽然也是一个session的leader，但一般它不会创建新的进程组，也没有job的管理功能，所以这种情况下一个session就只有一个进程组，所有的进程都属于同样的进程组和session。

我们这里看一下shell作为session leader的情况，假设我们在shell里面执行了这些命令：
```bash
dev@debian:~$ sleep 1000 &
[1] 1646
dev@debian:~$ cat | wc -l &
[2] 1648
dev@debian:~$ jobs
[1]-  Running                 sleep 1000 &
[2]+  Stopped                 cat | wc -l
```

下面这张图标明了这种情况下它们之间的关系：
```
+--------------------------------------------------------------+
|                                                              |
|      pg1             pg2             pg3            pg4      |
|    +------+       +-------+        +-----+        +------+   |
|    | bash |       | sleep |        | cat |        | jobs |   |
|    +------+       +-------+        +-----+        +------+   |
| session leader                     | wc  |                   |
|                                    +-----+                   |
|                                                              |
+--------------------------------------------------------------+
                            session
```

>pg = process group(进程组)

* bash是session的leader，sleep、cat、wc和jobs这四个进程都由bash fork而来，所以他们也属于这个session
* bash也是自己所在进程组的leader
* bash会为自己启动的每个进程都创建一个新的进程组，所以这里sleep和jobs进程属于自己单独的进程组
* 对于用管道符号“|”连接起来的命令，bash会将它们放到一个进程组中

## nohup
nohup是咋回事呢？nohup干了这么几件事：

* 将stdin重定向到/dev/null，于是程序读标准输入将会返回EOF
* 将stdout和stderr重定向到nohup.out或者用户通过参数指定的文件，程序所有输出到stdout和stderr的内容将会写入该文件（有时在文件中看不到输出，有可能是程序没有调用flush）
* 屏蔽掉SIGHUP信号
* 调用exec启动指定的命令（nohup进程将会被新进程取代，但进程ID不变）

从上面nohup干的事可以看出，通过nohup启动的程序有这些特点：

* nohup程序不负责将进程放到后台，这也是为什么我们经常在nohup命令后面要加上符号“&”的原因
* 由于stdin、stdout和stderr都被重定向了，nohup启动的程序不会读写tty
* 由于stdin重定向到了/dev/null，程序读stdin的时候会收到EOF返回值
* nohup启动的进程本质上还是属于当前session的一个进程组，所以在当前shell里面可以通过jobs看到nohup启动的程序
* 当session leader退出后，该进程会收到SIGHUP信号，但由于nohup帮我们忽略了该信号，所以该进程不会退出
* 由于session leader已经退出，而nohup启动的进程属于该session，于是出现了一种情况，那就是通过nohup启动的这个进程组所在的session没有leader，这是一种特殊的情况，内核会帮我们处理这种特殊情况，这里就不再深入介绍

通过nohup，我们最后达到了就算session leader（一般是shell）退出后，进程还可以照常运行的目的。

## deamon
通过nohup，就可以实现让进程在后台一直执行的功能，为什么我们还要写deamon进程呢？

从上面的nohup的介绍中可以看出来，虽然进程是在后台执行，但进程跟当前session还是有着千丝万缕的关系，至少其父进程还是被session管着的，所以我们还是需要一个跟任何session都没有关系的进程来实现deamon的功能。实现deamon进程的大概步骤如下：

* 调用fork生成一个新进程，然后原来的进程退出，这样新进程就变成了孤儿进程，于是被init进程接收，这样新进程就和调用进程没有父子关系了。
* 调用setsid，创建新的session，新进程将成为新session的leader，同时该新session不和任何tty关联。
* 切换当前工作目录到其它地方，一般是切换到根目录，这样就取消了对原工作目录的引用，如果原工作目录是某个挂载点下面的目录，这样就不会影响该挂载点的卸载。
* 关闭一些从父进程继承过来而自己不需要的fd，避免不小心读写这些fd。
* 重定向stdin、stdout和stderr，避免读写它们出现错误。

## 参考

* [Processes](https://www.win.tue.nl/~aeb/linux/lk/lk-10.html)
* [Job Control](https://www.gnu.org/savannah-checkouts/gnu/libc/manual/html_node/Job-Control.html)
* [那些永不消逝的进程](https://www.ibm.com/developerworks/cn/linux/1702_zhangym_demo/index.html)