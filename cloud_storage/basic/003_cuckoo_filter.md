[上一篇介绍了bloomfilter](002_bloomfilter.md)，它的一个最大缺点就是不支持删除，这里将介绍的Cuckoo filter是一种支持删除的filter。

相对于bloomfilter，它的主要优点有：

1. 支持删除操作
2. 查询效率更高   
3. 更高的空间利用率 

## 实现

Cuckoo filter基于Cuckoo hashing，也是两个table加两个hash函数。

回顾一下[前面介绍的cuckoo hashing](001_cuckoo_hashing.md)，如果两个key出现冲突会怎么样？会将原来位置上的key挤掉，那么原来位置上的key去哪里呢？怎么知道它在另一个table中的位置呢？对于cuckoo hashing来说，这个不是问题，因为数据就存在cuckoo hashing里面，根据key和hash函数就可以算出它在另一个table中的位置。 但Cuckoo filter为了节约空间，不可能将key存储起来，那么当冲突发生的时候，原来位置上的key怎么知道它在另一个table中的位置呢？

### 插入

对于cuckoo hashing来说，两个hash函数是相互独立的，因为key会存储在cuckoo hashing中，所以根据key和两个hash函数可以随时算出key在两个table中的位置。
```
h1(x) = hash1(x)      <---- 这里x表示key
h2(x) = hash2(x)
```

对于cuckoo filter来说，它不存储key，但会存储一个fingerprint，好处是fingerprint相对于key来说占用的空间会小很多
```
h1(x) = hash1(x),
h2(x) = h1(x) ⊕ hash2(x’s fingerprint)
```

由于是异或操作，根据h2(x)和fingerprint可以得到h1(x)
```
h1(x) = h2(x) ⊕ hash2(x’s fingerprint)
```

所以当冲突发生的时候，根据当前位置的hash值（即当前位置的索引）和fingerprint就可以算出在另一table中的位置。

### 查找

根据插入时的算法，算出h1(x)和h2(x)，再去t1和t2的相应位置找是否有同样fingerprint的记录，如果没有，表示key一定不存在，否则key在DataSet中可能存在

### 删除

删除跟查找一样，根据插入时的算法，算出h1(x)和h2(x)，再去t1和t2的相应位置找是否有同样fingerprint的记录，如果有，删掉一条（fingerprint可能有重复）

## 注意事项

### 数据冲突

如果出现x1和x2，hash1(x1) == hash1(x2)，且它们的fingerprint一样，cuckoo filter没法区分出x1和x2是否是同一个key。它会在table里面存储两个一样的fingerprint。由于delete的时候只会删除一个fingerprint，所以不会出现正确性问题。

由于这个特点，就要求在使用cuckoo filter的时候，不要重复的往里面插入相同的数据

* 一方面由于重复的数据都放在同样的位置，且同一个位置的空间有限，会导致数据插入失败
* 重复插入需要同等数量的删除操作，否则就会出现数据不存在，但cuckoo filter报存在的情况，增加FPP

**特别注意：删除操作次数一定不能多于插入操作次数**，否则会导致数据准确性问题。如上面的x1和x2，如果插入它们之后连续调用两次删除x1的操作，那么接下来查找x2的时候就会报数据不存在，出现数据准确性错误。

### insert失败

跟bloomfilter相比，cuckoo filter可能会出现insert失败的情况，空间满了或者冲突过多（同cuckoo hashing一样），出现这种情况后，当前的cuckoo filter就不能再用了，由于没有保存key，所以没法进行rehash，只能调整cuckoo filter的参数，然后根据DataSet重新构建全新的cuckoo filter。而bloomfilter就没有这种情况，冲突多只会提高FPP，不会因为数据插入不进去而导致正确性问题。

## 最佳实践

关于bucket的最佳大小和fingerprint的最佳长度， [Cuckoo Filter论文](https://www.cs.cmu.edu/~dga/papers/cuckoo-conext2014.pdf)里给出了详细的推导过程以及测试结果，详情情况请参考论文。

### hash table每个bucket的大小

4是一个很合适的大小，大点的值会让table的占用率变高，但同时也需要更长的fingerprint的来避免冲突，造成整体空间消耗变大

### fingerprint的长度

fingerprint的长度跟数据规模n以及bucket的大小有关，在bucket大小为4的情况下，2^15 ~ 2^30的数据量只需4 ~ 6 bits

## 优点分析

在文章的最开始说了cuckoo filter的三个优点，上面只分析了第一个优点，支持删除（update=delete+insert），这里简单分析一下其它的两个优点

### 查询效率更高 

如果应用场景需要很低的FPP，那么bloomfilter需要采用较大的table和较多的hash函数，但cuckoo filter始终只需要计算两次hash

* cuckoo filter的hash计算少，速度快
* cuckoo filter只会读两个位置的数据，而bloomfilter里面bit位可能比较分散，需要读多个位置的数据。

### 空间利用率高

bloomfilter对于每个hash结果的位置只需做一个标记，只占用1 bit的空间，而cuckoo filter需要记录fingerprint，明显需要更多的空间，为什么说cuckoo filter的空间利用率高呢？

因为在应用场景需要很低FPP的时候（< %3），bloomfilter由于冲突几率高，需要更大的bit数组长度来满足要求，但cuckoo filter由于冲突低，只需要较短的table即能满足要求。

所以虽然cuckoo filter中每个元素占用的空间大，但由于table的长度短，所以在当FPP低于一定值的时候（理论值是3%），cuckoo filter的空间利用率要高于bloomfilter

## 结论

通过上面的分析可以看出，cuckoo filter并不是在所有的场合都好于bloomfilter，当有大量删除操作的时候，cuckoo filter肯定要优于bloomfilter，但如果对FPP的要求不是很高，bloomfilter就不需要太多的hash函数和很长的table，无论从性能还是空间利用率都要优于cuckoo filter，同时bloomfilter的容错性较好，反复插入同样的数据不会影响正确性，同时也不需要关心删除操作。

## 参考

* [Cuckoo Filter: Practically Better Than Bloom](https://www.cs.cmu.edu/~dga/papers/cuckoo-conext2014.pdf)
    