### Swift 数据模型

图1

当第一次使用Swift时，主要包含下面几个步骤

  1. 创建一个新帐号（如何创建帐号跟采用的认证系统有关，详细信息见Tempauth或者Keystone）
  2. 用新帐号登录，创建container
  3. 然后就可以在对应的container下面创建文件了
    
当从服务器上取一个文件的时候，必须告诉该文件属于哪个帐号的哪个container，不然服务器不知道到哪里拿这个文件。

### Swift主要的server

* Proxy Server
    * session管理，帐号管理认证（Tempauth）
    * 分发客户的请求到各种服务器，比如将创建container的请求转发到container Server， 将创建对象的服务转发到Object Server

* Object Server
 
   存储具体的对象，也就是我们常说的文件

* Container Server

   存储object 到 container的对应关系，即每个container里面都有哪些object

* Account Server

   存储container 到 account的对应关系，即每个account里面都有哪些container

* storage Server

   Object Server， Container Server， Account Server的统称

### 各个server之间的关系
可以将所有的服务器都放在同一台机器上， 也可以分开放，他们之间在物理上没有必然的联系。
但通常情况下， 一般将proxy server放在一个单独的节点， 将所有的storage server放到另一个节点（因为Container Server， Account Server占用的空间较小，放在单独的机器上浪费空间。）， 于是一般情况下的网络拓扑图如下：

图2

这里的device可以理解为一块磁盘或者一个挂载点，所以一般情况下一台机器有多个磁盘或者挂载点， 就有多少个device。

### 登录服务器

* 帐号认证

    curl -i https://storage.clouddrive.com/v1/auth -H "X-Auth-User: jdoe" -H "X-Auth-Key: jdoepassword"
    * https://storage.clouddrive.com是proxy server所在机器的URL
    * jdoe是帐号
    * jdoepassword是密码
    
* 返回
    ```
    HTTP/1.1 204 No Content
    Date: Mon, 12 Nov 2010 15:32:21
    Server: Apache
    X-Storage-Url: $publicURL
    X-Auth-Token: $token
    Content-Length: 0
    Content-Type: text/plain; charset=UTF-8
    ```
    
    * $publicURL： 后续所有操作要访问的URL。
    既然前面登陆的时候已经输入了proxy server的URL，这里为什么还要返回一个URL呢，直接用前面的URL不就可以了？ 这个主要用来做负载均衡， 一般情况下一个系统有多台proxy server， 登陆的时候系统会返回一台负载相对较小的proxy server回来。如果系统只有一台proxy server，那这里返回的URL应该和你登录的时候指定的一样，比如都是https://storage.clouddrive.com。 Swift 根据什么样的策略做负载均衡？ 待研究。
    * $token： 相当于session id， 服务器用这个ID来检查当前会话是否已认证过

### 上传一个文件

curl –X PUT -i -H "X-Auth-Token: $token" -T ./file1 $publicURL/container/file1
 
* 首先proxy server收到文件上传请求，在请求的URL中含有account、container和object名称
* proxy server会根据account/containner/file1计算出一个4字节的hash code
* 取hash code的后n位（n<=32），在object ring里面找对应的partition和devices（一个device包含多个partition）

    比如在创建ring的时候指定的replica为3份， 那么这里会得到3个devices。 如果整个系统的devices数量不够3个，比如说只有2个， 那么这里只返回2个devices，也就是说只存2份replica。（在同台机器的同一挂载点放两份数据意义不大）
    
* 根据account/containner计算container的hash code，并根据hash code的后m位（m<=32），在container ring里面找对应的partition和devices（container的partition和object的partition是相互独立的，没有什么关系）
* 依次跟每个device所在的object server建立http连接，URL里面包含/object device/object partition/account/containner/file1， HTTP HEADER信息里面包含container所在的partition和device（一个连接里面包含container的一个device，这样不同的object server会更新不同container server上的数据）。

    如果创建container ring的时候指定的replica数目是4， 创建object ring的时候指定的replica数目是3， 那么这里container的devices数目和object的devices数目不一样，会不会导致只更新了3个container replica的信息，还有一个没更新到？ 有待考证
    
* 依次接收客户端的数据，并转发给每个device上的object server。（如果两个device在一台机器上，会往同一台机器上传两份，这种情况只出现在系统里面只有两台object server的情况下）
* object server收到请求后，会将文件保存在指定device上的partition对应的目录下
* object server根据http HEAD里的信息，连接container所在device上的container server， 并请求更新container， HTTP请求中包含container的partition和device信息。

    这里的请求如果失败，会纪录下来，由updater进程后续去更新。
    
* container server收到请求后，会更新指定device上的partition对应的目录下的container信息，使其包含这个新建的object
* proxy server等待所有的object server完成相应的工作，如果发现有大于(n//2)+1个replica写入成功（这里的n是总的待写入的replica的数目），就会返回成功。（如果有1个失败了咋办？ 见后续介绍）

### 创建container   
创建contianer和上传文件的原理是一样的，也是根据hash code在container ring里面找对应的partition和devices， 然后跟每个device上的container server建立http连接并转发相关的信息。

### 删除object   
删除object和创建object的流程是一样的，也是根据object名称找到partition和不同的devices，然后依次删掉它们，和创建不同的地方是不需要传输文件。

### 总结
从上面可以看出，只要弄懂了ring，以及partition、device的概念，其他的地方都很好理解。
