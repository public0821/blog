###ring的作用

swift中有3个ring，他们分别是account ring， container ring和object ring, 他们的数据结构和计算方式完全一样，但他们之间是相互独立的， 没有任何关系。 

* account ring
    根据account name可以查找到对应的account信息存放在哪个device
* container ring
    根据container name可以查找到对应的container信息存放在哪个device
* object ring
    根据object name可以查找到对应的ring信息存放在哪个device
    
###ring的结构

官方网站和其他很多文章都都从ring的角度介绍swift的ring， 在我个人来看， ring更像一个table，这里会以一个table的角度来介绍ring（个人觉得更直观）。

这里有几个概念需要明确一下

* replica
    就是文件备份，有几个replica就表示对于每个文件，系统中会存几分
* 节点
    就是我们常说的一台服务器，他上面会挂很多硬盘
* device
    可以理解为实际的硬盘或者虚拟的硬盘，一个节点上可以有很多device
* partition
    是一个逻辑的抽象概念，没有具体的实物与之对应，可以理解为文件夹，相当于在系统里面创建了很多很多个文件夹，然后将他们均匀的放到不同的device上。

上传一个文件的流程（假设replica是3）：
* proxy服务器收到上传请求，然后根据/account/container/object这样一串名字计算hash码。
* 取hash码的后n位，得到的数字就是partition的id， 一个partition会放在三个不同的device上，以防某一份丢失后导致数据丢失。
* 根据partition id从ring里面找到对应的3个device。
* 依次将文件上传到各个device所在的节点上去
* 节点上的object server收到请求后，就会将文件存在相应device的相应partition中，可以理解为放到相应磁盘的相应目录下去。（具体存储的目录结构和格式会在后续介绍到）

###ring的生成

ring的生成和调整由命令swift-ring-builder完成， 关于swift-ring-builder的具体用法见相关的文档。

我们常用的场景主要是添加或者删除device，然后rebalance整个ring。 运行命令前，请确保先备份原来的ring文件。 运行完之后需要手动将相应的ring文件拷贝到其他的所有机器上。

注意： swift-ring-builder只是用来操纵ring文件， ring文件发生变化后， 相应object的存放位置变化由其他的机制来触发（后续会介绍到）, 这里的rebalance只是重新调整ring文件，不对系统中存储的内容做任何改动。

swift-ring-builder object.builder create 20 3 1
swift-ring-builder object.builder add r1z1-127.0.0.1:6010/sdb1 1
swift-ring-builder object.builder add r1z1-127.0.0.1:6010/sdb2 1
swift-ring-builder object.builder add r1z2-127.0.0.2:6010/sdb1 2
swift-ring-builder object.builder add r1z3-127.0.0.2:6010/sdb2 2
swift-ring-builder object.builder add r1z2-127.0.0.3:6010/sdb1 1
swift-ring-builder object.builder add r1z3-127.0.0.3:6010/sdb2 1
swift-ring-builder object.builder rebalance
swift-ring-builder container.builder create 16 3 1
swift-ring-builder container.builder add r1z1-127.0.0.1:6011/sdb1 1
swift-ring-builder container.builder add r1z1-127.0.0.1:6011/sdb2 1
swift-ring-builder container.builder add r1z2-127.0.0.2:6011/sdb1 2
swift-ring-builder container.builder add r1z3-127.0.0.2:6011/sdb2 2
swift-ring-builder container.builder rebalance
swift-ring-builder account.builder create 8 3 1
swift-ring-builder account.builder add r1z1-127.0.0.1:6011/sdb1 1
swift-ring-builder account.builder add r1z1-127.0.0.1:6011/sdb2 1
swift-ring-builder account.builder add r1z2-127.0.0.2:6011/sdb1 2
swift-ring-builder account.builder add r1z3-127.0.0.2:6011/sdb2 2
swift-ring-builder account.builder rebalance

上面命令的参数意义请参考swift-ring-builder的使用文档。这里有几点需要注意：

* 权重主要跟存储容量有关

    比如1T的硬盘设置权重为1， 2T的硬盘设置权重为2，这样swift就会往2T的硬盘中存两倍于1T硬盘的数据， 如果1T和2T的硬盘都设置为1，造成的后果就是1T的盘存满了，2T的盘才用了一半。
    
* account，container和object是不同的ring，他们生成的文件也是单独分开的
* 往ring里面加的是device， 不是整个节点， 也就是说需要一块硬盘一块硬盘的往ring里面加， 没法一次把一个节点上的所有磁盘加进去。
* 同一节点上的device用同样的ip和端口（如果使用不用的端口会怎么样？想不出这么做有什么好处，待研究）， 不同节点的device可以用不同的端口
* 每个ring的part_power可以不一样， 但从数据均匀分布的角度来看，最好设置一样。

    * 比如现在有20个device，每个object存3份， 将part_power设置成8，就有2^8*3=192个partition，意味着这个系统支持的最大device数为192， 如果以后device数量增长到200，就会有至少8个device上不会有数据，处于闲置状态，所以设置part_power的时候不仅要考虑现有的device数量，还要考虑以后可能扩展到的device数量。
    * 如果account ring的part_power为2，每个account存3份， 则account的partition数目为2^2*3=12， 则account的信息最多会放到12个device上，当系统的device大于12的时候，除这12个device外，其他的device上就不会存放account信息，于是这12个device上的数据就会比其他device的多， 如果account信息很多，则会造成数据在各个device上分布的不均匀。
    
* 每个ring的replica数目理论上可以不一样（没有仔细研究），比如account和container的replica数量可以设置的多点，丢失的可能性更低。

###ring生成的算法

###rebalance的算法    
    
