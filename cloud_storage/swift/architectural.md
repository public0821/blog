###Swift 数据模型

图1

当使用Swift时，主要包含下面几个步骤

  1. 创建一个新帐号（如何创建帐号跟采用的认证系统有关，详细信息见Tempauth或者Keystone）
  2. 用新帐号登录，创建container (见RESTful API)
  3. 然后就可以在对应的container下面创建、删除文件了 (见RESTful API)
    
对文件进行操作的时候，必须告诉Swift该文件属于哪个帐号的哪个container，不然它不知道到哪台服务器上操作该文件。

###Swift主要的Server

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

   负责接收Proxy server转发过来的Account操作请求，比如往它里面添加、删除Container，根据请求的内容更新对应Account DB里面的内容，一般一个node上只有一个Account Server

    注意：Account Server不支持创建和删除账号操作，所有的账号都由认证系统管理（Tempauth或者Keystone）。那么如何清空一个用户下的所有文件呢？（待研究）

* Storage Server

   它是个逻辑概念，不存在具体的这么一个进程，是Object Server， Container Server， Account Server的统称，由于一般情况下这三个Server都放在同一个node上，所以也可以将Storage Server理解成一个存放这三个Server的node

###各个Server之间的关系

* 可以将所有的Server都放在同一台机器上， 也可以分开放，他们之间在物理上没有必然的联系。
* 但通常情况下， 一般将Proxy Server放在一个单独的node， 将所有的Storage Server放到另一个node（因为Container Server， Account Server占用的空间较小，放在单独的机器上浪费空间。）

###其它的Server

* Replication
* Updaters
* Auditors
