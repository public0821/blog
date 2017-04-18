# netfilter/iptables简介

netfilter和iptables是什么关系？常说的iptables里面的表(table)、链(chain)、规则(rule)都是什么东西？本篇将带着这些疑问介绍netfilter/iptables的结构和相关概念，帮助有需要的同学更好的理解netfilter/iptables，为进一步学习使用iptables做准备。

## 什么是netfilter和iptables
用通俗点的话来讲:

* netfilter指整个项目，不然官网就不会叫www.netfilter.org了。
* 在这个项目里面，netfilter特指内核中的netfilter框架，iptables指用户空间的配置工具。
* netfilter在协议栈中添加了5个钩子，允许内核模块在这些钩子的地方注册回调函数，这样经过钩子的所有数据包都会被注册在相应钩子上的函数所处理，包括修改数据包内容、给数据包打标记或者丢掉数据包等。
* netfilter框架负责维护钩子上注册的处理函数或者模块，以及它们的优先级。
* iptables是用户空间的一个程序，通过netlink和内核的netfilter框架打交道，负责往钩子上配置回调函数。
* netfilter框架负责在需要的时候动态加载其它的内核模块，比如 ip_conntrack、nf_conntrack、NAT subsystem等。

>在应用者的眼里，可能iptables代表了整个项目，代表了防火墙，但在开发者眼里，可能netfilter更能代表这个项目。

## netfilter钩子（hooks）
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

>关于这些钩子更具体的位置，请参考[Linux网络数据包的接收过程](https://segmentfault.com/a/1190000008836467)和[数据包的发送过程](https://segmentfault.com/a/1190000008926093)

从上面的流程中，我们还可以看出，不考虑特殊情况的话，一个数据包只会经过下面三个路径中的一个：

* 本机收到目的IP是本机的数据包: NF_IP_PRE_ROUTING -> NF_IP_LOCAL_IN
* 本机收到目的IP不是本机的数据包: NF_IP_PRE_ROUTING -> NF_IP_FORWARD -> NF_IP_POST_ROUTING
* 本机发出去的数据包: NF_IP_LOCAL_OUT -> NF_IP_POST_ROUTING

>注意： netfilter所有的钩子（hooks）都是在内核协议栈的IP层，由于IPv4和IPv6用的是不同的IP层代码，所以iptables配置的rules只会影响IPv4的数据包，而IPv6相关的配置需要使用ip6tables。

## iptables中的表（tables）
iptables用表（table）来分类管理它的规则（rule），根据rule的作用分成了好几个表，比如用来过滤数据包的rule就会放到filter表中，用于处理地址转换的rule就会放到nat表中，其中rule就是应用在netfilter钩子上的函数，用来修改数据包的内容或过滤数据包。目前iptables支持的表有下面这些：

#### Filter
从名字就可以看出，这个表里面的rule主要用来过滤数据，用来控制让哪些数据可以通过，哪些数据不能通过，它是最常用的表。

#### NAT
里面的rule都是用来处理网络地址转换的，控制要不要进行地址转换，以及怎样修改源地址或目的地址，从而影响数据包的路由，达到连通的目的，这是家用路由器必备的功能。

#### Mangle
里面的rule主要用来修改IP数据包头，比如修改TTL值，同时也用于给数据包添加一些标记，从而便于后续其它模块对数据包进行处理（这里的添加标记是指往内核skb结构中添加标记，而不是往真正的IP数据包上加东西）。

#### Raw
在netfilter里面有一个叫做connection tracking的功能（后面会介绍到），主要用来追踪所有的连接，而raw表里的rule的功能是给数据包打标记，从而控制哪些数据包不被connection tracking所追踪。

#### Security
里面的rule跟SELinux有关，主要是在数据包上设置一些SELinux的标记，便于跟SELinux相关的模块来处理该数据包。

## chains
上面我们根据不同功能将rule放到了不同的表里面之后，这些rule会注册到哪些钩子上呢？于是iptables将表中的rule继续分类，让rule属于不同的链（chain），由chain来决定什么时候触发chain上的这些rule。

iptables里面有5个内置的chains，分别对应5个钩子：

* PREROUTING: 数据包经过NF_IP_PRE_ROUTING时会触发该chain上的rule.
* INPUT: 数据包经过NF_IP_LOCAL_IN时会触发该chain上的rule.
* FORWARD: 数据包经过NF_IP_FORWARD时会触发该chain上的rule.
* OUTPUT: 数据包经过NF_IP_LOCAL_OUT时会触发该chain上的rule.
* POSTROUTING: 数据包经过NF_IP_POST_ROUTING时会触发该chain上的rule.

每个表里面都可以包含多个chains，但并不是每个表都能包含所有的chains，因为某些表在某些chain上没有意义或者有些多余，比如说raw表，它只有在connection tracking之前才有意义，所以它里面包含connection tracking之后的chain就没有意义。（connection tracking的位置会在后面介绍到）

多个表里面可以包含同样的chain，比如在filter和raw表里面，都有OUTPUT chain，那应该先执行哪个表的OUTPUT chain呢？这就涉及到后面会介绍的优先级的问题。

>提示：可以通过命令```iptables -L -t nat|grep policy|grep Chain```查看到nat表所支持的chain，其它的表也可以用类似的方式查看到，比如修改nat为raw即可看到raw表所支持的chain。

## 每个表（table）都包含哪些chain，表之间的优先级是怎样的？
下图在上面那张图的基础上，详细的标识出了各个表的rule可以注册在哪个钩子上（即各个表里面支持哪些chain），以及它们的优先级。


> 1. 图中每个钩子关联的表按照优先级高低，从上到下排列；
> 2. 图中将nat分成了SNAT和DNAT，便于区分；
> 3. 图中标出了connection tracking（可以简单的把connection tracking理解成一个不能配置chain和rule的表，它必须放在指定位置，只能enable和disable）。


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


* 以NF_IP_PRE_ROUTING为例，数据包到了这个点之后，会先执行raw表中PREROUTING(chain)里的rule，然后执行connection tracking，接着再执行mangle表中PREROUTING(chain)里的rule，最后执行nat (DNAT)表中PREROUTING(chain)里的rule。
* 以filter表为例，它只能注册在NF_IP_LOCAL_IN、NF_IP_FORWARD和NF_IP_LOCAL_OUT上，所以它只支持INPUT、FORWARD和OUTPUT这三个chain。
* 以收到目的IP是本机的数据包为例，它的传输路径为：NF_IP_PRE_ROUTING -> NF_IP_LOCAL_IN，那么它首先要依次经过NF_IP_PRE_ROUTING上注册的raw、connection tracking 、mangle和nat (DNAT)，然后经过NF_IP_LOCAL_IN上注册的mangle、filter、security和nat (SNAT)。

## iptables中的规则（Rules）
rule存放在特定表的特定chain上，每条rule包含下面两部分信息：

#### Matching
Matching就是如何匹配一个数据包，匹配条件很多，比如协议类型、源/目的IP、源/目的端口、in/out接口、包头里面的数据以及连接状态等，这些条件可以任意组合从而实现复杂情况下的匹配。详情请参考[Iptables matches](https://www.frozentux.net/iptables-tutorial/iptables-tutorial.html#MATCHES)

#### Targets
Targets就是找到匹配的数据包之后怎么办，常见的有下面几种：

* DROP：直接将数据包丢弃，不再进行后续的处理
* RETURN： 跳出当前chain，该chain里后续的rule不再执行
* QUEUE： 将数据包放入用户空间的队列，供用户空间的程序处理
* ACCEPT： 同意数据包通过，继续执行后续的rule
* 跳转到其它用户自定义的chain继续执行

当然iptables包含的targets很多很多，但并不是每个表都支持所有的targets，
rule所支持的target由它所在的表和chain以及所开启的扩展功能来决定，具体每个表支持的targets请参考[Iptables targets and jumps](https://www.frozentux.net/iptables-tutorial/iptables-tutorial.html#TARGETS)。

## 用户自定义Chains
除了iptables预定义的5个chain之外，用户还可以在表中定义自己的chain，用户自定义的chain中的rule和预定义chain里的rule没有区别，不过由于自定义的chain没有和netfilter里面的钩子进行绑定，所以它不会自动触发，只能从其它chain的rule中跳转过来。

## 连接追踪（Connection Tracking）
Connection Tracking发生在NF_IP_PRE_ROUTING和NF_IP_LOCAL_OUT这两个地方，一旦开启该功能，Connection Tracking模块将会追踪每个数据包（被raw表中的rule标记过的除外），维护所有的连接状态，然后这些状态可以供其它表中的rule引用，用户空间的程序也可以通过/proc/net/ip_conntrack来获取连接信息。下面是所有的连接状态：

>这里的连接不仅仅是TCP的连接，两台设备的进程用UDP和ICMP（ping）通信也会被认为是一个连接

* NEW: 当检测到一个不和任何现有连接关联的新包时，如果该包是一个合法的建立连接的数据包（比如TCP的sync包或者任意的UDP包），一个新的连接将会被保存，并且标记为状态NEW。
* ESTABLISHED: 对于状态是NEW的连接，当检测到一个相反方向的包时，连接的状态将会由NEW变成ESTABLISHED，表示连接成功建立。对于TCP连接，意味着收到了一个SYN/ACK包， 对于UDP和ICMP，任何反方向的包都可以。
* RELATED: 数据包不属于任何现有的连接，但它跟现有的状态为ESTABLISHED的连接有关系，对于这种数据包，将会创建一个新的连接，且状态被标记为RELATED。这种连接一般是辅助连接，比如FTP的数据传输连接（FTP有两个连接，另一个是控制连接），或者和某些连接有关的ICMP报文。
* INVALID: 数据包不和任何现有连接关联，并且不是一个合法的建立连接的数据包，对于这种连接，将会被标记为INVALID，一般这种都是垃圾数据包，比如收到一个TCP的RST包，但实际上没有任何相关的TCP连接，或者别的地方误发过来的ICMP包。
* UNTRACKED: 被raw表里面的rule标记为不需要tracking的数据包，这种连接将会标记成UNTRACKED。

## 结束语
本篇介绍了netfilter/iptables的基本结构和一些相关概念，了解这些只是一个开始，要想熟练的配置iptables规则，还需要对计算机网络知识非常熟悉，下次有机会的话再介绍如何使用iptables。

## 参考

* [Iptables Tutorial](https://www.frozentux.net/iptables-tutorial/iptables-tutorial.html)
* [A Deep Dive into Iptables and Netfilter Architecture](https://www.digitalocean.com/community/tutorials/a-deep-dive-into-iptables-and-netfilter-architecture)
