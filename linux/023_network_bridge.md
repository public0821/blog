#Linux虚拟网络设备之bridge(桥)

继前两篇介绍了[tun/tap](https://segmentfault.com/a/1190000009249039)和[veth](https://segmentfault.com/a/1190000009251098)之后，本篇将介绍Linux下非常常用的一种虚拟网络设备，那就是bridge(桥)。

本篇将通过实际的例子来一步一步解释bridge是如何工作的。

## 什么是bridge？
首先，bridge是一个虚拟网络设备，所以具有网络设备的特征，可以配置IP、MAC地址等；其次，bridge是一个虚拟交换机，和物理交换机有类似的功能。

对于普通的网络设备来说，只有两端，从一端进来的数据会从另一端出去，如物理网卡从外面网络中收到的数据会转发给内核协议栈，而从协议栈过来的数据会转发到外面的物理网络中。

而bridge不同，bridge有多个端口，数据可以从任何端口进来，进来之后从哪个口出去和物理交换机的原理差不多，要看mac地址。

## 创建bridge
我们先用iproute2创建一个bridge：
```bash
dev@debian:~$ sudo ip link add name br0 type bridge
dev@debian:~$ sudo ip link set br0 up
```

当刚创建一个bridge时，它是一个独立的网络设备，只有一个端口连着协议栈，其它的端口啥都没连，这样的bridge没有任何实际功能，如下图所示：
```
+----------------------------------------------------------------+
|                                                                |
|       +------------------------------------------------+       |
|       |             Newwork Protocol Stack             |       |
|       +------------------------------------------------+       |
|              ↑                                ↑                |
|..............|................................|................|
|              ↓                                ↓                |
|        +----------+                     +------------+         |
|        |   eth0   |                     |     br0    |         |
|        +----------+                     +------------+         |
| 192.168.3.21 ↑                                                 |
|              |                                                 |
|              |                                                 |
+--------------|-------------------------------------------------+
               ↓
         Physical Network
```

>这里假设eth0是我们的物理网卡，IP地址是192.168.3.21，网关是192.168.3.1

## 将bridge和veth设备相连
创建一对veth设备，并配置上IP
```bash
dev@debian:~$ sudo ip link add veth0 type veth peer name veth1
dev@debian:~$ sudo ip addr add 192.168.3.101/24 dev veth0
dev@debian:~$ sudo ip addr add 192.168.3.102/24 dev veth1
dev@debian:~$ sudo ip link set veth0 up
dev@debian:~$ sudo ip link set veth1 up
```

将veth0连上bridge
```bash
dev@debian:~$ sudo ip link set dev veth0 master br0
#通过bridge link命令可以看到bridge上连接了哪些设备
dev@debian:~$ sudo bridge link
6: veth0 state UP : <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 master br0 state forwarding priority 32 cost 2
```

这时候，网络就变成了这个样子:
```
+----------------------------------------------------------------+
|                                                                |
|       +------------------------------------------------+       |
|       |             Newwork Protocol Stack             |       |
|       +------------------------------------------------+       |
|            ↑            ↑              |            ↑          |
|............|............|..............|............|..........|
|            ↓            ↓              ↓            ↓          |
|        +------+     +--------+     +-------+    +-------+      |
|        | .3.21|     |        |     | .3.101|    | .3.102|      |
|        +------+     +--------+     +-------+    +-------+      |
|        | eth0 |     |   br0  |<--->| veth0 |    | veth1 |      |
|        +------+     +--------+     +-------+    +-------+      |
|            |                           ↑            ↑          |
|            |                           |            |          |
|            |                           +------------+          |
|            |                                                   |
+------------|---------------------------------------------------+
             ↓
     Physical Network
```

>这里为了画图方便，省略了IP地址前面的192.168，比如.3.21就表示192.168.3.21

bridge和veth0相连之后，发生了两个变化：

* bridge和veth0之间连接起来了，并且是双向的通道
* 协议栈和veth0之间变成了单通道，协议栈能发数据给veth0，但veth0从外面收到的数据不会转发给协议栈

相当于bridge在veth0和协议栈之间插了一脚，在veth0上面做了点小动作，将veth0本来要转发给协议栈的数据给拦截了，全部转发给bridge了，同时bridge也可以向veth0发数据。

下面来检验一下是不是这样的：

通过veth0 ping veth1失败：
```bash
dev@debian:~$ ping -c 1 -I veth0 192.168.3.102
PING 192.168.2.1 (192.168.2.1) from 192.168.2.11 veth0: 56(84) bytes of data.
From 192.168.2.11 icmp_seq=1 Destination Host Unreachable

--- 192.168.2.1 ping statistics ---
1 packets transmitted, 0 received, +1 errors, 100% packet loss, time 0ms
```

为什么veth0加入了bridge之后，就ping不通veth2了呢？ 先抓包看看：
```bash
#由于veth0的arp缓存里面没有veth1的mac地址，所以ping之前先发arp请求
#从veth1上抓包来看，veth1收到了arp请求，并且返回了应答
dev@debian:~$ sudo tcpdump -n -i veth1
tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
listening on veth1, link-type EN10MB (Ethernet), capture size 262144 bytes
21:43:48.353509 ARP, Request who-has 192.168.3.102 tell 192.168.3.101, length 28
21:43:48.353518 ARP, Reply 192.168.3.102 is-at 26:58:a2:57:37:e9, length 28

#从veth0上抓包来看，数据包也发出去了，并且也收到了返回
dev@debian:~$ sudo tcpdump -n -i veth0
tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
listening on veth0, link-type EN10MB (Ethernet), capture size 262144 bytes
21:44:09.775392 ARP, Request who-has 192.168.3.102 tell 192.168.3.101, length 28
21:44:09.775400 ARP, Reply 192.168.3.102 is-at 26:58:a2:57:37:e9, length 28

#再看bridge上的数据包，发现只有应答
dev@debian:~$ sudo tcpdump -n -i br0
tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
listening on br0, link-type EN10MB (Ethernet), capture size 262144 bytes
21:45:48.225459 ARP, Reply 192.168.3.102 is-at 26:58:a2:57:37:e9, length 28
```
从上面的抓包可以看出，去和回来的流程都没有问题，问题就出在veth0收到应答包后没有给协议栈，而是给了bridge，于是协议栈得不到veth1的mac地址，从而通信失败。

## 给bridge配上IP
通过上面的分析可以看出，给veth0配置IP没有意义，因为就算协议栈传数据包给veth0，应答包也回不来。这里我们就将veth0的IP让给bridge。
```bash
dev@debian:~$ sudo ip addr del 192.168.3.101/24 dev veth0
dev@debian:~$ sudo ip addr add 192.168.3.101/24 dev br0
```

于是网络变成了这样子：
```
+----------------------------------------------------------------+
|                                                                |
|       +------------------------------------------------+       |
|       |             Newwork Protocol Stack             |       |
|       +------------------------------------------------+       |
|            ↑            ↑                           ↑          |
|............|............|...........................|..........|
|            ↓            ↓                           ↓          |
|        +------+     +--------+     +-------+    +-------+      |
|        | .3.21|     | .3.101 |     |       |    | .3.102|      |
|        +------+     +--------+     +-------+    +-------+      |
|        | eth0 |     |   br0  |<--->| veth0 |    | veth1 |      |
|        +------+     +--------+     +-------+    +-------+      |
|            |                           ↑            ↑          |
|            |                           |            |          |
|            |                           +------------+          |
|            |                                                   |
+------------|---------------------------------------------------+
             ↓
     Physical Network
```
> 其实veth0和协议栈之间还是有联系的，但由于veth0没有配置IP，所以协议栈在路由的时候不会将数据包发给veth0，就算强制要求数据包通过veth0发送出去，但由于veth0另一端收到的数据包只会给br0，所以协议栈还是没法收到相应的arp应答包，导致通信失败。
> 这里为了表达更直观，将协议栈和veth0之间的联系去掉了，veth0相当于一根网线。

再通过br0 ping一下veth1，结果成功
```bash
dev@debian:~$ ping -c 1 -I br0 192.168.3.102
PING 192.168.3.102 (192.168.3.102) from 192.168.3.101 br0: 56(84) bytes of data.
64 bytes from 192.168.3.102: icmp_seq=1 ttl=64 time=0.121 ms

--- 192.168.3.102 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 0.121/0.121/0.121/0.000 ms
```

但ping网关还是失败，因为这个bridge上只有两个网络设备，分别是192.168.3.101和192.168.3.102，br0不知道192.168.3.1在哪。
```bash
dev@debian:~$ ping -c 1 -I br0 192.168.3.1
PING 192.168.3.1 (192.168.3.1) from 192.168.3.101 br0: 56(84) bytes of data.
From 192.168.3.101 icmp_seq=1 Destination Host Unreachable

--- 192.168.3.1 ping statistics ---
1 packets transmitted, 0 received, +1 errors, 100% packet loss, time 0ms
```

## 将物理网卡添加到bridge
将eth0添加到br0上：
```bash
dev@debian:~$ sudo ip link set dev eth0 master br0
dev@debian:~$ sudo bridge link
2: eth0 state UP : <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 master br0 state forwarding priority 32 cost 4
6: veth0 state UP : <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 master br0 state forwarding priority 32 cost 2
#dev@debian:~$ sudo brctl stp br0 on
```

br0根本不区分接入进来的是物理设备还是虚拟设备，对它来说都一样的，都是网络设备，所以当eth0加入br0之后，落得和上面veth0一样的下场，从外面网络收到的数据包将无条件的转发给br0，自己变成了一根网线。

这时通过eth0来ping网关失败，但由于br0通过eth0这根网线连上了外面的物理交换机，所以连在br0上的设备都能ping通网关，这里连上的设备就是veth1和br0自己，veth1是通过veth0这根网线连上去的，而br0可以理解为自己有一块自带的网卡。
```bash
#通过eth0来ping网关失败
dev@debian:~$ ping -c 1 -I eth0 192.168.3.1
PING 192.168.3.1 (192.168.3.1) from 192.168.3.21 eth0: 56(84) bytes of data.
From 192.168.3.21 icmp_seq=1 Destination Host Unreachable

--- 192.168.3.1 ping statistics ---
1 packets transmitted, 0 received, +1 errors, 100% packet loss, time 0ms

#通过br0来ping网关成功
dev@debian:~$ ping -c 1 -I br0 192.168.3.1
PING 192.168.3.1 (192.168.3.1) from 192.168.3.101 br0: 56(84) bytes of data.
64 bytes from 192.168.3.1: icmp_seq=1 ttl=64 time=27.5 ms

--- 192.168.3.1 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 27.518/27.518/27.518/0.000 ms

#通过veth1来ping网关成功
dev@debian:~$ ping -c 1 -I veth1 192.168.3.1
PING 192.168.3.1 (192.168.3.1) from 192.168.3.102 veth1: 56(84) bytes of data.
64 bytes from 192.168.3.1: icmp_seq=1 ttl=64 time=68.8 ms

--- 192.168.3.1 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 68.806/68.806/68.806/0.000 ms
```

由于eth0已经变成了和网线差不多的功能，所以在eth0上配置IP已经没有什么意义了，并且还会影响协议栈的路由选择，比如如果上面ping的时候不指定网卡的话，协议栈有可能优先选择eth0，导致ping不通，所以这里需要将eth0上的IP去掉。
```bash
#在本人的测试机器上，由于eth0上有IP，
#访问192.168.3.0/24网段时，会优先选择eth0
dev@debian:~$ sudo route -v
Kernel IP routing table
Destination     Gateway         Genmask         Flags Metric Ref    Use Iface
default         192.168.3.1     0.0.0.0         UG    0      0        0 eth0
link-local      *               255.255.0.0     U     1000   0        0 eth0
192.168.3.0     *               255.255.255.0   U     0      0        0 eth0
192.168.3.0     *               255.255.255.0   U     0      0        0 veth1
192.168.3.0     *               255.255.255.0   U     0      0        0 br0

#由于eth0已结接入了br0，所有它收到的数据包都会转发给br0，
#于是协议栈收不到应答包，导致ping失败
dev@debian:~$ ping -c 1 192.168.3.1
PING 192.168.3.1 (192.168.3.1) 56(84) bytes of data.
From 192.168.3.21 icmp_seq=1 Destination Host Unreachable

--- 192.168.3.1 ping statistics ---
1 packets transmitted, 0 received, +1 errors, 100% packet loss, time 0ms

#将eth0上的IP删除掉
dev@debian:~$ sudo ip addr del 192.168.3.21/24 dev eth0

#再ping一次，成功
dev@debian:~$ ping -c 1 192.168.3.1
PING 192.168.3.1 (192.168.3.1) 56(84) bytes of data.
64 bytes from 192.168.3.1: icmp_seq=1 ttl=64 time=3.91 ms

--- 192.168.3.1 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 3.916/3.916/3.916/0.000 ms

#这是因为eth0没有IP之后，路由表里面就没有它了，于是数据包会从veth1出去
dev@debian:~$ sudo route -v
Kernel IP routing table
Destination     Gateway         Genmask         Flags Metric Ref    Use Iface
192.168.3.0     *               255.255.255.0   U     0      0        0 veth1
192.168.3.0     *               255.255.255.0   U     0      0        0 br0
#从这里也可以看出，由于原来的默认路由走的是eth0，所以当eth0的IP被删除之后，
#默认路由不见了，想要连接192.168.3.0/24以外的网段的话，需要手动将默认网关加回来

#添加默认网关，然后再ping外网成功
dev@debian:~$ sudo ip route add default via 192.168.3.1
dev@debian:~$ ping -c 1 baidu.com
PING baidu.com (111.13.101.208) 56(84) bytes of data.
64 bytes from 111.13.101.208: icmp_seq=1 ttl=51 time=30.6 ms

--- baidu.com ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 30.690/30.690/30.690/0.000 ms

```

经过上面一系列的操作后，网络变成了这个样子：
```
+----------------------------------------------------------------+
|                                                                |
|       +------------------------------------------------+       |
|       |             Newwork Protocol Stack             |       |
|       +------------------------------------------------+       |
|                         ↑                           ↑          |
|.........................|...........................|..........|
|                         ↓                           ↓          |
|        +------+     +--------+     +-------+    +-------+      |
|        |      |     | .3.101 |     |       |    | .3.102|      |
|        +------+     +--------+     +-------+    +-------+      |
|        | eth0 |<--->|   br0  |<--->| veth0 |    | veth1 |      |
|        +------+     +--------+     +-------+    +-------+      |
|            |                           ↑            ↑          |
|            |                           |            |          |
|            |                           +------------+          |
|            |                                                   |
+------------|---------------------------------------------------+
             ↓
     Physical Network
```

## bridge必须要配置IP吗？
在我们常见的物理交换机中，有可以配置IP和不能配置IP两种，不能配置IP的交换机一般通过com口连上去做配置（更简单的交换机连com口的没有，不支持任何配置），而能配置IP的交换机可以在配置好IP之后，通过该IP远程连接上去做配置，从而更方便。

bridge就属于后一种交换机，自带虚拟网卡，可以配置IP，该虚拟网卡一端连在bridge上，另一端跟协议栈相连。和物理交换机一样，bridge的工作不依赖于该虚拟网卡，即没有给这个网卡配置IP，bridge也能照样工作。

删除br0上的IP:
```bash
dev@debian:~$ sudo ip addr del 192.168.3.101/24 dev br0
```

于是网络变成了这样子，相当于br0的一个端口通过eth0连着交换机，另一个端口通过veth0连着veth1：
```
+----------------------------------------------------------------+
|                                                                |
|       +------------------------------------------------+       |
|       |             Newwork Protocol Stack             |       |
|       +------------------------------------------------+       |
|                                                     ↑          |
|.....................................................|..........|
|                                                     ↓          |
|        +------+     +--------+     +-------+    +-------+      |
|        |      |     |        |     |       |    | .3.102|      |
|        +------+     +--------+     +-------+    +-------+      |
|        | eth0 |<--->|   br0  |<--->| veth0 |    | veth1 |      |
|        +------+     +--------+     +-------+    +-------+      |
|            |                           ↑            ↑          |
|            |                           |            |          |
|            |                           +------------+          |
|            |                                                   |
+------------|---------------------------------------------------+
             ↓
     Physical Network
```


## 参考

* [Virtual networking in Linux](https://www.ibm.com/developerworks/library/l-virtual-networking/)
* [Linux 上的基础网络设备详解](https://www.ibm.com/developerworks/cn/linux/1310_xiawc_networkdevice/)