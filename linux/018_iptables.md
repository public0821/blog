# netfilter/iptables简介

本篇将介绍netfilter/iptables的结构和相关概念，帮助有需要的同学理解netfilter/iptables，为进一步更好的学习配置iptables规则做准备。

## 什么是netfilter和iptables
* netfilter在Linux内核协议栈中设置了一些钩子，并且netfilter允许内核模块在这些钩子的地方注册回调函数，这样经过协议栈中钩子的所有数据包都会被注册在相应钩子上的函数所处理，包括修改数据包内容、给数据包打标记或者丢掉数据包等。

* iptables是一个应用层程序，通过netlink和内核的netfilter框架打交道，负责配置netfilter支持的rule。iptables也可以指netfilter支持的功能，
iptables is a generic table structure for the definition of rulesets. Each rule within an IP table consists of a number of classifiers (iptables matches) and one connected action (iptables target).

我们常说的netfilter是指内核中完成所有这些功能的框架，它主要由netfilter、ip_tables、 ip_conntrack、nf_conntrack、NAT subsystem这及部分构成

这是[官网](https://www.netfilter.org/)的介绍:

## netfilter hooks
在内核协议栈中，有5个跟netfilter有关的钩子，数据包经过每个钩子时，都会检查上面是否注册有函数，如果有的话，就会调用相应的函数处理该数据包，它们的位置见下图：
```
         |
         | Incoming
         ↓
+-------------------+
| NF_IP_PRE_ROUTING |
+-------------------+
         |
         |
         ↓
+------------------+
|                  |         +----------------+
| routing decision |-------->| NF_IP_LOCAL_IN |
|                  |         +----------------+
+------------------+                 |
         |                           |
         |                           ↓
         |                  +-----------------+
         |                  | local processes |
         |                  +-----------------+
         |                           |
         |                           |
         ↓                           ↓
 +---------------+          +-----------------+
 | NF_IP_FORWARD |          | NF_IP_LOCAL_OUT |
 +---------------+          +-----------------+
         |                           |
         |                           |
         ↓                           |
+------------------+                 |
|                  |                 |
| routing decision |<----------------+
|                  |
+------------------+
         |
         |
         ↓
+--------------------+
| NF_IP_POST_ROUTING |
+--------------------+
         |
         | Outgoing
         ↓
```

* NF_IP_PRE_ROUTING: 接收的数据包刚进入协议栈的IP层，还没有经过路由选择，即还不知道数据包是要发给本机还是其它机器。
* NF_IP_LOCAL_IN: 已经经过路由选择，并且该数据包的目的IP是本机，进入本地数据包处理流程。
* NF_IP_FORWARD: 已经经过路由选择，但该数据包的目的IP不是本机，而是其它机器，进入forward流程。
* NF_IP_LOCAL_OUT: 本地程序要发出去的数据包刚到IP层，还没进行路由选择。
* NF_IP_POST_ROUTING: 本地程序发出去的数据包，或者转发（forward）的数据包已经经过了路由选择，即将交由下层发送出去。

>关于这些钩子更具体的位置，请参考Linux网络[数据包的接收过程]和[数据包的发送过程]

从上面的流程中，我们还可以看出，不考虑特殊情况的话，一个数据包只会经过下面三个路径中的一个：

* 发往本机且目的IP是本机的数据包: NF_IP_PRE_ROUTING -> NF_IP_LOCAL_IN
* 发往本机但目的IP不是本机的数据包: NF_IP_PRE_ROUTING -> NF_IP_FORWARD -> NF_IP_POST_ROUTING
* 本机发出去的数据包: NF_IP_LOCAL_OUT -> NF_IP_POST_ROUTING

>由于每个钩子的位置都可以注册很多函数，所以他们需要有一个优先级，即优先执行哪个，后面会有介绍。

## iptables tables
iptables用tables来管理它的rules，根据rule的作用和目的分成了好几个table，而rule就是应用在netfilter钩子上的函数，来修改数据包的内容或过滤数据包。目前iptables支持的tables有下面这些：

#### Filter
从名字就可以看出，这个table里面的rule主要用来过滤数据，用来控制让哪些数据可以通过，哪些数据不能通过，它是最常用的table。

#### NAT
里面的rule都是用来进行网络地址转换的，控制要不要进行地址转换，以及怎样修改源地址或目的地址，从而影响数据包的路由，达到连通的目的，这是家用路由器必备的功能。

#### Mangle
里面的rule主要用来修改IP数据包头，比如修改TTL值，同时也用于给数据包添加一些标记，从而便于后续其它工具对数据包进行处理（这里的添加标记是指往内核skb结构中添加标记，而不是真正的IP数据包上加东西）。

#### Raw
在netfilter里面有一个叫做connection tracking的功能，主要用来追踪所有的连接，而raw table里的rule的功能是控制哪些数据包不被connection tracking所追踪。

#### Security
里面的rule跟SELinux有关，主要是在数据包上设置一些SELinux的标记，便于跟SELinux相关的模块来处理该数据包。

## chains
我们有了上面根据不同功能rules分类的表，

## 参考

* [A Deep Dive into Iptables and Netfilter Architecture](https://www.digitalocean.com/community/tutorials/a-deep-dive-into-iptables-and-netfilter-architecture)


INVALID: Packets can be marked INVALID if they are not associated with an existing connection and aren't appropriate for opening a new connection, if they cannot be identified, or if they aren't routable among other reasons.

垃圾数据，比如收到一个tcp的RST，但根本就没有想过的connection