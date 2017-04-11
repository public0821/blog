# netfilter/iptables简介

本篇将介绍netfilter/iptables的结构和相关概念，帮助有需要的同学理解netfilter/iptables，为进一步更好的学习配置iptables规则做准备。

## 什么是netfilter和iptables
这是[官网](https://www.netfilter.org/)的介绍:

* netfilter在Linux内核协议栈中设置了一些钩子，并且netfilter允许内核模块在这些钩子的地方注册回调函数，这样经过协议栈中钩子的所有数据包都会被注册在相应钩子上的函数所处理，包括修改数据包内容、给数据包打标记或者丢掉数据包等。

* iptables是一个通用的用来定义规则库（rulesets）的结构，table里面的每条规则由一系列的匹配条件和一个最终的动作构成（即怎么匹配，匹配了之后做什么事）。（这里iptables的概念是iptables里面tables的概念，而不是我们常说的iptables工具）

上面的介绍比较官方，下面用通俗点的话来解释一下：

* netfilter指整个项目，不然官网就不会叫www.netfilter.org了。
* 进入这个项目后，netfilter特指内核中的netfilter框架，iptables指应用层的配置工具。
* netfilter在协议栈中添加了5个钩子，netfilter框架负责维护钩子上注册的处理函数或者模块，以及它们的优先级。
* iptables通过netlink和netfilter框架打交道，负责配置哪些功能（模块）应该注册到哪个钩子位置。
* netfilter框架负责在需要的时候动态加载其它的内核模块，比如 ip_conntrack、nf_conntrack、NAT subsystem等。
* iptables里面的tables就是按照功能（模块）进行分类的rule的集合。

>在应用者的眼里，iptables代表了整个项目，代表了防火墙，但在开发者眼里，netfilter更能代表这个项目。

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

* NF_IP_PRE_ROUTING: 接收的数据包刚进来，还没有经过路由选择，即还不知道数据包是要发给本机还是其它机器。
* NF_IP_LOCAL_IN: 已经经过路由选择，并且该数据包的目的IP是本机，进入本地数据包处理流程。
* NF_IP_FORWARD: 已经经过路由选择，但该数据包的目的IP不是本机，而是其它机器，进入forward流程。
* NF_IP_LOCAL_OUT: 本地程序要发出去的数据包刚到IP层，还没进行路由选择。
* NF_IP_POST_ROUTING: 本地程序发出去的数据包，或者转发（forward）的数据包已经经过了路由选择，即将交由下层发送出去。

>关于这些钩子更具体的位置，请参考Linux网络[数据包的接收过程](https://segmentfault.com/a/1190000008836467)和[数据包的发送过程](https://segmentfault.com/a/1190000008926093)

从上面的流程中，我们还可以看出，不考虑特殊情况的话，一个数据包只会经过下面三个路径中的一个：

* 本机收到目的IP是本机的数据包: NF_IP_PRE_ROUTING -> NF_IP_LOCAL_IN
* 本机收到目的IP不是本机的数据包: NF_IP_PRE_ROUTING -> NF_IP_FORWARD -> NF_IP_POST_ROUTING
* 本机发出去的数据包: NF_IP_LOCAL_OUT -> NF_IP_POST_ROUTING

>注意： netfilter所有的hooks都是在内核协议栈的IP层，由于IPv4和IPv6用的是不同的IP层代码，所以iptables配置的rules只会影响IPv4的数据包，而IPv6相关的配置需要使用ip6tables。

## iptables tables
iptables用tables来分类管理它的rules，根据rule的作用分成了好几个table，而rule就是应用在netfilter钩子上的函数，用来修改数据包的内容或过滤数据包。目前iptables支持的tables有下面这些：

#### Filter
从名字就可以看出，这个table里面的rule主要用来过滤数据，用来控制让哪些数据可以通过，哪些数据不能通过，它是最常用的table。

#### NAT
里面的rule都是用来进行网络地址转换的，控制要不要进行地址转换，以及怎样修改源地址或目的地址，从而影响数据包的路由，达到连通的目的，这是家用路由器必备的功能。

#### Mangle
里面的rule主要用来修改IP数据包头，比如修改TTL值，同时也用于给数据包添加一些标记，从而便于后续其它工具对数据包进行处理（这里的添加标记是指往内核skb结构中添加标记，而不是往真正的IP数据包上加东西）。

#### Raw
在netfilter里面有一个叫做connection tracking的功能，主要用来追踪所有的连接，而raw table里的rule的功能是控制哪些数据包不被connection tracking所追踪。

#### Security
里面的rule跟SELinux有关，主要是在数据包上设置一些SELinux的标记，便于跟SELinux相关的模块来处理该数据包。

## chains
上面我们根据不同功能将rule放到了不同的table里面之后，这些rule会注册到哪些钩子上呢？于是iptables将table中的rules继续分类，让rule属于不同的chain，由chain来决定什么时候触发chain上的这些rule。

iptables里面有5个内置的chains，分别对应5个钩子：

* PREROUTING: 数据包经过NF_IP_PRE_ROUTING时会触发该chain上的rule.
* INPUT: 数据包经过NF_IP_LOCAL_IN时会触发该chain上的rule.
* FORWARD: 数据包经过NF_IP_FORWARD时会触发该chain上的rule.
* OUTPUT: 数据包经过NF_IP_LOCAL_OUT时会触发该chain上的rule.
* POSTROUTING: 数据包经过NF_IP_POST_ROUTING时会触发该chain上的rule.

每个table里面都可以包含多个chains，但并不是每个table都能包含所有的chains，因为某些table在某些chain上没有意义或者有些多余，比如说raw table，它只有在connection tracking之前才有意义，所以它里面包含connection tracking之后的chain就没有意义。

多个table里面可以包含同样的chain，比如在filter和raw table里面，都有OUTPUT chain，那应该先执行哪个table的OUTPUT chain呢？这就涉及到后面会介绍的优先级的问题。

>提示：可以通过命令```iptables -L -t nat|grep policy|grep Chain```查看到nat所支持的chain，其它的table也可以用类似的方式查看到。

## 每个table都包含哪些chain，table间的优先级是怎样的？
下图在上面那张图的基础上，详细的标识出了各个table的规则可以注册在哪个钩子上（即各个table里支持哪些chain），以及它们的优先级：

```
                                    |
                                    | Incoming             ++---------------------++
                                    ↓                      || raw                 ||
                           +-------------------+           || connection tracking ||
                           | NF_IP_PRE_ROUTING |= = = = = =|| mangle              ||
                           +-------------------+           || nat (DNAT)          ||
                                    |                      ++---------------------++
                                    |
                                    ↓                                                ++------------++
                           +------------------+                                      || mangle     ||
                           |                  |         +----------------+           || filter     ||
                           | routing decision |-------->| NF_IP_LOCAL_IN |= = = = = =|| security   ||
                           |                  |         +----------------+           || nat (SNAT) ||
                           +------------------+                 |                    ++------------++
                                    |                           |
                                    |                           ↓
                                    |                  +-----------------+
                                    |                  | local processes |
                                    |                  +-----------------+
                                    |                           |
                                    |                           |                    ++---------------------++
 ++------------++                   ↓                           ↓                    || raw                 ||
 || mangle     ||           +---------------+          +-----------------+           || connection tracking ||
 || filter     ||= = = = = =| NF_IP_FORWARD |          | NF_IP_LOCAL_OUT |= = = = = =|| mangle              ||
 || security   ||           +---------------+          +-----------------+           || nat (DNAT)          ||
 ++------------++                   |                           |                    || filter              ||
                                    |                           |                    || security            ||
                                    ↓                           |                    ++---------------------++
                           +------------------+                 |
                           |                  |                 |
                           | routing decision |<----------------+
                           |                  |
                           +------------------+
                                    |
                                    |
                                    ↓
                           +--------------------+           ++------------++
                           | NF_IP_POST_ROUTING |= = = = = =|| mangle     ||
                           +--------------------+           || nat (SNAT) ||
                                    |                       ++------------++
                                    | Outgoing
                                    ↓
```

>图中每个钩子关联的table列表的优先级都是从上到下，同时也将nat分成了SNAT和DNAT，便于区分，并且标出了connection tracking（可以简单的把connection tracking理解成一个不能配置chain和rule的table，它必须放在指定位置，只能enable和disable）。

* 以filter为例，它只能注册在NF_IP_LOCAL_IN、NF_IP_FORWARD和NF_IP_LOCAL_OUT上，所以它只支持INPUT、FORWARD和OUTPUT这三个chain。
* 以收到目的IP是本机的数据包为例，它的传输路径为：NF_IP_PRE_ROUTING -> NF_IP_LOCAL_IN，那么它首先要依次经过NF_IP_PRE_ROUTING上注册的raw、connection tracking 、mangle和nat (DNAT)，然后经过NF_IP_LOCAL_IN上注册的mangle、filter、security和nat (SNAT)。

## IPTables Rules
rule存放在特定table的特定chain上，每条rule包含下面两部分信息：

#### Matching
Matching就是如何匹配一个数据包，匹配条件很多，比如协议类型、源/目的IP、源/目的端口、in/out接口、包头里面的数据以及连接状态等，这些条件可以任意组合从而实现复杂情况下的匹配。详情请参考[Iptables matches](https://www.frozentux.net/iptables-tutorial/iptables-tutorial.html#MATCHES)

#### Targets
Targets就是找到匹配的数据包之后怎么办，常见的有下面几种：

* DROP：直接将数据包丢弃，不再进行后续的处理
* RETURN： 跳出当前chain，该chain里后续的rule不再执行
* QUEUE： 将数据包放入用户空间的队列，供用户空间的程序处理
* ACCEPT： 同意数据包通过，继续执行后续的rule
* 跳转到其它用户自定义的chain继续执行

当然iptables包含的targets很多很多，但并不是每个table都支持所有的targets，
rule所支持的target由它所在的table和chain以及所开启的扩展功能来决定，具体每个table支持的rule targets请参考[Iptables targets and jumps](https://www.frozentux.net/iptables-tutorial/iptables-tutorial.html#TARGETS)。

## 用户自定义Chains
除了iptables预定义的5个chain之外，用户还可以在table中自定义自己的chain，用户自定义的chain由于没有和netfilter里面的钩子进行绑定，所以它不会自动触发，只能从其它chain的rule中跳转过来。

## Connection Tracking

### Available States

## 参考

* [Iptables Tutorial](https://www.frozentux.net/iptables-tutorial/iptables-tutorial.html)
* [A Deep Dive into Iptables and Netfilter Architecture](https://www.digitalocean.com/community/tutorials/a-deep-dive-into-iptables-and-netfilter-architecture)


INVALID: Packets can be marked INVALID if they are not associated with an existing connection and aren't appropriate for opening a new connection, if they cannot be identified, or if they aren't routable among other reasons.

垃圾数据，比如收到一个tcp的RST，但根本就没有想过的connection