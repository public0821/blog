# Linux Namespace系列（06）：network namespace (CLONE_NEWNET)

network namespace用来隔离网络设备, IP地址, 端口等. 每个namespace将会有自己独立的网络栈，路由表，防火墙规则，socket等。

每个新的network namespace默认有一个本地环回接口，除了lo接口外，所有的其他网络设备（物理/虚拟网络接口，网桥等）只能属于一个network namespace。每个socket也只能属于一个network namespace。     

当新的network namespace被创建时，lo接口默认是关闭的，需要自己手动启动起

标记为"local devices"的设备不能从一个namespace移动到另一个namespace，比如loopback, bridge, ppp等，我们可以通过ethtool -k命令来查看设备的netns-local属性。
```bash
#这里“on”表示该设备不能被移动到其他network namespace
dev@ubuntu:~$ ethtool -k lo|grep netns-local
netns-local: on [fixed]
```

>本篇所有例子都在ubuntu-server-x86_64 16.04下执行通过

##示例

本示例将演示如何创建新的network namespace并同外面的namespace进行通信。

```bash
#--------------------------第一个shell窗口----------------------
#记录默认network namespace ID
dev@ubuntu:~$ readlink /proc/$$/ns/net
net:[4026531957]

#创建新的network namespace
dev@ubuntu:~$ sudo unshare --uts --net /bin/bash
root@ubuntu:~# hostname container001
root@ubuntu:~# exec bash
root@container001:~# readlink /proc/$$/ns/net
net:[4026532478]

#运行ifconfig啥都没有
root@container001:~# ifconfig
root@container001:~#

#启动lo （这里不详细介绍ip这个tool的用法，请参考man ip）
root@container001:~# ip link set lo up
root@container001:~# ifconfig
lo        Link encap:Local Loopback
          inet addr:127.0.0.1  Mask:255.0.0.0
          inet6 addr: ::1/128 Scope:Host
          UP LOOPBACK RUNNING  MTU:65536  Metric:1
          RX packets:0 errors:0 dropped:0 overruns:0 frame:0
          TX packets:0 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:1
          RX bytes:0 (0.0 B)  TX bytes:0 (0.0 B)

root@container001:~# ping 127.0.0.1
PING 127.0.0.1 (127.0.0.1) 56(84) bytes of data.
64 bytes from 127.0.0.1: icmp_seq=1 ttl=64 time=0.070 ms
64 bytes from 127.0.0.1: icmp_seq=2 ttl=64 time=0.015 ms

#获取当前bash进程的PID
root@container001:~# echo $$
15812

#--------------------------第二个shell窗口----------------------
#创建新的虚拟以太网设备，让两个namespace能通讯
dev@ubuntu:~$ sudo ip link add veth0 type veth peer name veth1

#将veth1移动到上面第一个窗口中的namespace
#这里15812是上面bash的PID
dev@ubuntu:~$ sudo ip link set veth1 netns 15812

#为veth0分配IP并启动veth0
dev@ubuntu:~$ sudo ip address add dev veth0 192.168.8.1/24
dev@ubuntu:~$ sudo ip link set veth0 up
dev@ubuntu:~$ ifconfig veth0
veth0     Link encap:Ethernet  HWaddr 9a:4d:d5:96:b5:36
          inet addr:192.168.8.1  Bcast:0.0.0.0  Mask:255.255.255.0
          inet6 addr: fe80::984d:d5ff:fe96:b536/64 Scope:Link
          UP BROADCAST RUNNING MULTICAST  MTU:1500  Metric:1
          RX packets:8 errors:0 dropped:0 overruns:0 frame:0
          TX packets:8 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:1000
          RX bytes:648 (648.0 B)  TX bytes:648 (648.0 B)

#--------------------------第一个shell窗口----------------------
#为veth1分配IP地址并启动它
root@container001:~# ip address add dev veth1 192.168.8.2/24
root@container001:~# ip link set veth1 up
root@container001:~# ifconfig veth1
veth1     Link encap:Ethernet  HWaddr 6a:dc:59:79:3c:8b
          inet addr:192.168.8.2  Bcast:0.0.0.0  Mask:255.255.255.0
          inet6 addr: fe80::68dc:59ff:fe79:3c8b/64 Scope:Link
          UP BROADCAST RUNNING MULTICAST  MTU:1500  Metric:1
          RX packets:8 errors:0 dropped:0 overruns:0 frame:0
          TX packets:8 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:1000
          RX bytes:648 (648.0 B)  TX bytes:648 (648.0 B)

#连接成功
root@container001:~# ping 192.168.8.1
PING 192.168.8.1 (192.168.8.1) 56(84) bytes of data.
64 bytes from 192.168.8.1: icmp_seq=1 ttl=64 time=0.098 ms
64 bytes from 192.168.8.1: icmp_seq=2 ttl=64 time=0.023 ms
```

到目前为止，两个namespace之间可以网络通信了，但在container001里还是不能访问外网。下面将通过NAT的方式让container001能够上外网。这部分内容完全是网络相关的知识，跟namespace已经没什么关系了。

```bash
#--------------------------第二个shell窗口----------------------
#回到上面示例中的第二个窗口

#确认IP forward是否已经开通，这里1表示开通了
#如果你的机器上是0，请运行这个命令将它改为1： sudo sysctl -w net.ipv4.ip_forward=1
dev@ubuntu:~$ cat /proc/sys/net/ipv4/ip_forward
1

#添加NAT规则，这里ens32是机器上连接外网的网卡
#关于iptables和nat都比较复杂，这里不做解释
dev@ubuntu:~$ sudo iptables -t nat -A POSTROUTING -o ens32 -j MASQUERADE

#--------------------------第一个shell窗口----------------------
#回到第一个窗口，添加默认网关
root@container001:~# ip route add default via 192.168.8.1
root@container001:~# route -n
Kernel IP routing table
Destination     Gateway         Genmask         Flags Metric Ref    Use Iface
0.0.0.0         192.168.8.1     0.0.0.0         UG    0      0        0 veth1
192.168.8.0     0.0.0.0         255.255.255.0   U     0      0        0 veth1

#这样就可以访问外网了
#由于测试环境的限制，所以采用下面的方式检测网络是否畅通
#如果网络没有什么限制的话，随便ping一个外部的IP测试就可以了
root@container001:~# curl -I www.google.com
HTTP/1.1 200 OK
Date: Fri, 15 Jul 2016 08:12:03 GMT

```

network namespace的概念比较简单，但如何做好网络的隔离和连通却比较难，包括性能和安全相关的考虑，需要很好的Linux网络知识。后续在介绍docker网络管理的时候会对Linux网络做一个更详细的介绍。

##参考
[Namespaces in operation, part 7: Network namespaces](https://lwn.net/Articles/580893/)