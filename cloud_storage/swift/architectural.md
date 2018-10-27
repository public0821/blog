### Swift 数据模型

图1

当使用Swift时，主要包含下面几个步骤

  1. 创建一个新帐号（如何创建帐号跟采用的认证系统有关，详细信息见Tempauth或者Keystone）
  2. 用新帐号登录，创建container (见RESTful API)
  3. 然后就可以在对应的container下面创建、删除文件了 (见RESTful API)
    
对文件进行操作的时候，必须告诉Swift该文件属于哪个帐号的哪个container，不然它不知道到哪台服务器上操作该文件。

### Swift主要的Server

这里的Server不是指具体的服务器机器，而是我们常说的进程，一个node(指具体的一台服务器机器)上可以放多个Server

图2
* Proxy Server
    * 存储系统的入口，所有的client都和它打交道
    * session管理，帐号认证管理（可选，如Tempauth）
    * 分发客户的请求到各种服务器，比如将创建container的请求转发到container Server， 将创建对象的服务转发到Object Server
    * 上传和读取文件时，所有的流量都要经过它，但它不缓存任何文件
    * 由于一般情况下Proxy Server会单独的部署在一个node上，所以也可以将它看成一个node

* Object Server
 
   负责接收Proxy server转发过来的文件操作请求，然后将文件存储在本地相应的磁盘上，一般情况下一个node有很多块磁盘，但只有一个Object Server

* Container Server

   负责接收Proxy server转发过来的Container操作请求，比如创建、删除Container, 或者往它里面添加、删除Object，根据请求的内容更新对应Container DB里面的内容，一般一个node上只有一个Container Server

* Account Server

   负责接收Proxy server转发过来的Account操作请求（比如添加或者删除帐号，或者往它里面添加、删除Container）， 根据请求的内容更新对应Account DB里面的内容，一般一个node上只有一个Account Server

    注意：Account Server不负责帐号的认证。 只负责管理Account里面的Container。

* Storage Server

   它是个逻辑概念，不存在具体的这么一个进程，是Object Server， Container Server， Account Server的统称，由于一般情况下这三个Server都放在同一个node上，所以也可以将Storage Server理解成一个存放这三个Server的node

### 各个Server之间的关系

* 可以将所有的Server都放在同一台机器上， 也可以分开放，他们之间在物理上没有必然的联系。
* 但通常情况下， 一般将Proxy Server放在一个单独的node， 将所有的Storage Server放到另一个node（因为Container Server， Account Server占用的空间较小，放在单独的机器上浪费空间。）

### 其它的Server

* Replication

    * 遍历本地的partition，找到本地partition对应的其他server（比如系统设置的是3份备份，则本地的每个partition都能找到其它的两个node，他们上面也放有相同的partition），然后和其他node上的partition比较，如果自己比对方新，则将自己新的文件同步给对方。（当自己的文件比对方旧时，什么都不做，因为他不知道要拉对方的什么数据过来）
    * 删除掉标记为删除的object, container, account

* Updaters

    * swift-object-updater： 当有object创建的时候，需要更新相应的container，让新建的object包含在对应的container里面，这里的更新操作有时可能失败，比如网络问题或者Container Server很忙，这个时候系统会将更新操作记录在磁盘上，等待Updater Server后续继续更新Account Server。(同样适用于删除object)
    * swift-container-updater： 同swift-object-updater类似
    * swift-account-updater： 没有，因为增加和删除account的时候，不需要更新任何东西


* Auditors

  swift-account-auditor，swift-container-auditor，swift-object-auditor： 定期检查本地节点上相应数据，如果发现数据有损坏（如磁盘出现坏道，人为修改文件等），则将有问题的数据隔离，等待Replication Server同步正常的数据过来
