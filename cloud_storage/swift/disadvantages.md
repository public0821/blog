
### Swift 的缺点

* 没有机制自动控制数据均匀分布

    如果数据足够多的话，理论上hash(account/container/object)和文件大小应该是均匀分布的，但如果出现偏差怎么办，比如刚好一个node上放了很多上G的文件，把空间占满了，虽然其他的node还有大量的空间，但也需要对这个node做手工调整，不然下次hash值再落到这个node上的文件将不会被存储。

* 扩容不太方便

    扩容需要移动数据，比如原来是5个节点，现在要再增加5个节点，那么需要移动一半的数据，当容量非常大时，持续的时间很长，并且对整个系统会造成很大的压力。
    
* Region功能较弱

    比如有两个Region，我希望存储的数据为当前Region存两份，另一个Region存一份。 访问不同的Region会从对应的Region拿数据
    
* 不支持Erasure Code

    在对性能要求不高的情况下，EC能大大的提高存储效率
    
* 不支持策略配置

    Swift不支持在Account或者Container上设置存储策略。 一个云存储系统可能用于多种用途，比如在一个公司内部，有经常访问的数据和不经常访问的备份数据，他们不希望为了这两个不同的需求搭建两套存储系统。
