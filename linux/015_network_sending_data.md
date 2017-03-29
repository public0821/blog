#Linux网络 - 数据包的发送过程

继上一篇介绍了[数据包的接收过程]后，本文将介绍在Linux系统中，数据包是如何一步一步从应用程序到网卡并最终发送出去的。

如果英文没有问题，强烈建议阅读后面参考里的文章，里面介绍的更详细。

>本文只讨论以太网的物理网卡，并且以一个UDP包的发送过程作为示例

## socket层
```
               +-------------+
               | Application |
               +-------------+
                     |
                     |
                     ↓
+------------------------------------------+
| socket(AF_INET, SOCK_DGRAM, IPPROTO_UDP) |
+------------------------------------------+
                     |
                     |
                     ↓
           +-------------------+
           | sendto(sock, ...) |
           +-------------------+
                     |
                     |
                     ↓
              +--------------+
              | inet_sendmsg |
              +--------------+
                     |
                     |
                     ↓
             +---------------+
             | inet_autobind |
             +---------------+
                     |
                     |
                     ↓
               +-----------+
               | UDP layer |
               +-----------+


```
* **socket(...)：** 创建一个socket结构体，并初始化相应的操作函数，由于我们定义的是UDP的socket，所以里面存放的都是跟UDP相关的函数
* **sendto(sock, ...)：** 应用层的程序（Application）调用该函数开始发送数据包，该函数数会调用后面的inet_sendmsg
* **inet_sendmsg：** 该函数主要是检查当前socket有没有绑定源端口，如果没有的话，调用inet_autobind分配一个，然后调用UDP层的函数
* **inet_autobind：** 该函数会调用socket上绑定的get_port函数获取一个可用的端口，由于该socket是UDP的socket，所以get_port函数会调到UDP代码里面的相应函数。

## UDP层
```
                     |
                     |
                     ↓
              +-------------+
              | udp_sendmsg |
              +-------------+
                     |
                     |
                     ↓
          +----------------------+
          | ip_route_output_flow |
          +----------------------+
                     |
                     |
                     ↓
              +-------------+
              | ip_make_skb |
              +-------------+
                     |
                     |
                     ↓
         +------------------------+
         | udp_send_skb(skb, fl4) |
         +------------------------+
                     |
                     |
                     ↓
                +----------+
                | IP layer |
                +----------+
~

```
* **udp_sendmsg：** udp模块发送数据包的入口，该函数较长，在该函数中会先调用ip_route_output_flow获取路由信息（主要包括源IP和网卡），然后调用ip_make_skb构造skb结构体，最后将网卡的信息和该skb关联。
* **ip_route_output_flow：** 该函数会根据路由表和目的IP，找到这个数据包应该从哪个设备发送出去，如果该socket没有绑定源IP，该函数还会根据路由表找到一个最合适的源IP给它。 如果该socket已经绑定了源IP，但根据路由表，从这个源IP对应的网卡没法到达目的地址，则该包会被丢弃，于是数据发送失败，sendto函数将返回错误。最后会将找到的设备和源IP塞进flowi4结构体并返回给udp_sendmsg
* **ip_make_skb：** 该函数的功能是构造skb包，构造好的skb包里面已经分配了IP包头，并且初始化了部分信息（IP包头的源IP就在这里被设置进去），同时该函数会调用__ip_append_dat，如果需要分片的话，会在__ip_append_data函数中进行分片，同时还会在该函数中检查socket的send buffer是否已经用光，如果被用光的话，返回ENOBUFS
* **udp_send_skb(skb, fl4)** 主要是往skb里面填充UDP的包头，同时处理checksum，然后调用IP层的相应函数。

## IP层
```
          |
          |
          ↓
   +-------------+
   | ip_send_skb |
   +-------------+
          |
          |
          ↓
  +-------------------+       +-------------------+       +---------------+
  | __ip_local_out_sk |------>| NF_INET_LOCAL_OUT |------>| dst_output_sk |
  +-------------------+       +-------------------+       +---------------+
                                                                  |
                                                                  |
                                                                  ↓
 +------------------+        +----------------------+       +-----------+
 | ip_finish_output |<-------| NF_INET_POST_ROUTING |<------| ip_output |
 +------------------+        +----------------------+       +-----------+
          |
          |
          ↓
  +-------------------+      +------------------+       +----------------------+
  | ip_finish_output2 |----->| dst_neigh_output |------>| neigh_resolve_output |
  +-------------------+      +------------------+       +----------------------+
                                                                   |
                                                                   |
                                                                   ↓
                                                           +----------------+
                                                           | dev_queue_xmit |
                                                           +----------------+
```
                                                           

* **ip_send_skb：** IP模块发送数据包的入口，该函数只是简单的调用一下后面的函数
* **__ip_local_out_sk：** 设置IP报文头的长度和checksum，然后调用下面netfilter的钩子
* **NF_INET_LOCAL_OUT：** netfilter的钩子，可以通过iptables来配置怎么处理该数据包，如果该数据包没被丢弃，则继续往下走
* **dst_output_sk：** 该函数根据skb里面的信息，调用相应的output函数，在我们UDP IPv4这种情况下，会调用ip_output
* **ip_output：** 将上面udp_sendmsg得到的网卡信息写入skb，然后调用NF_INET_POST_ROUTING的钩子
* **NF_INET_POST_ROUTING：** 在这里，用户有可能配置了SNAT或者，从而导致该skb的路由信息发生变化
* **ip_finish_output：** 这里会判断经过了上一步后，路由信息是否发生变化，如果发生变化的话，需要重新调用dst_output_sk（重新调用这个函数时，可能就不会再走到ip_output，而是走到被netfilter指定的output函数，这里有可能是xfrm4_transport_output），否则判断是否需要分片，如果需要就先做分片，否则直接往下走
* **ip_finish_output2：** 根据目的IP到路由表里面找到下一跳(nexthop)的地址，然后调用__ipv4_neigh_lookup_noref去arp表里面找下一跳的neigh信息，没找到的话会调用__neigh_create构造一个空的neigh结构体
* **dst_neigh_output：** 在该函数中，如果上一步ip_finish_output2没得到neigh信息，那么将会走到函数neigh_resolve_output中，否则直接调用neigh_hh_output，在该函数中，会将neigh信息里面的mac地址填到skb中，然后调用dev_queue_xmit发送数据包
* **neigh_resolve_output：** 该函数里面会发送arp请求，得到下一跳的mac地址，然后将mac地址填到skb中并调用dev_queue_xmit

## netdevice subsystem
```
                          |
                          |
                          ↓
                   +----------------+
  +----------------| dev_queue_xmit |
  |                +----------------+
  |                       |
  |                       |
  |                       ↓
  |              +-----------------+
  |              | Traffic Control |
  |              +-----------------+
  | loopback              |
  |   or                  +--------------------------------------------------------------+
  | IP tunnels            ↓                                                              |
  |                       ↓                                                              |
  |            +---------------------+  Failed   +----------------------+         +---------------+
  +----------->| dev_hard_start_xmit |---------->| raise NET_TX_SOFTIRQ |- - - - >| net_tx_action |
               +---------------------+           +----------------------+         +---------------+
                          |
                          +----------------------------------+
                          |                                  |
                          ↓                                  ↓
                  +----------------+              +------------------------+
                  | ndo_start_xmit |              | packet taps(AF_PACKET) |
                  +----------------+              +------------------------+
```
* **dev_queue_xmit：** 
* dev_hard_start_xmit： dev_hard_start_xmit返回失败后，会抛出软中断NET_TX_SOFTIRQ，当然如果是loopback或者IP tunnels的话，是不会抛中断，而是直接返回

## Device Driver
ndo_start_xmit会绑定到具体网卡驱动的相应函数，到这步之后，就归网卡驱动管了，不同的网卡驱动有不同的处理方式，大概的流程如下：
1. 将skb放入发送队列
2. 通知网卡发送数据包
3. 发送完成后进行skb的清理工作

## 参考

* [Monitoring and Tuning the Linux Networking Stack: Sending Data](https://blog.packagecloud.io/eng/2017/02/06/monitoring-tuning-linux-networking-stack-sending-data/)