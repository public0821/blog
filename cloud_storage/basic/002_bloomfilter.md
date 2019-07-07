## 作用

Bloomfilter主要用来快速判断某个元素是否在指定集合内，具有运行速度快，占用内存少的优点。

```
                                                     +-------------+
                                                     |             |
                                                     |             |
                           +-------------+           |             |
 +--------+     get/put    |             |           |             |
 | client | <------------->| bloomfilter |<--------->|  DataSet    |
 +--------+                |             |           |             |
                           +-------------+           |             |
                                                     |             |
                                                     |             |
                                                     +-------------+
```

如上图所示， bloomfilter处于data set和需要访问data set的client之间，client在对data set进行修改的时候通知bloomfilter，读取data set的时候先询问bloomfilter数据是否在data set中，如果不存在则立即返回，否则继续访问data set。

在需要bloomfilter的应用场景中，通常访问data set的开销比较大，比如data set存放在磁盘上或者网络上

## 实现

bloomfilter主要由两部分构成，存放数据的bit数组和处理数据的hash函数，数组的长度和hash函数的个数取决于data set的大小以及所期望的fpp值（后面介绍）。

### 插入

一般在往data set里面插入数据之前，先更新bloomfilter，这样可以保证只要数据成功写入data set，那么bloomfilter就一定已经更新成功，数据的正确性就得到了保证，不然就会出现数据写成功了，但bloomfilter没有更新成功的情况，导致下次读取数据的时候bloomfilter报数据不存在。

如何更新bloomfilter呢？我们这里假设采用两个hash函数fun1和fun2， bit数组的长度是10，依次插入两个数据d1和d2。

1. 用两个hash函数对d1进行hash，假设hash结果跟bit数组的长度取余之后为分别是2和5，那么将bit数组的第2和第5位设置位1
```
  0   1   2   3   4   5   6   7   8   9
+---+---+---+---+---+---+---+---+---+---+
| 0 | 0 | 1 | 0 | 0 | 1 | 0 | 0 | 0 | 0 |
+---+---+---+---+---+---+---+---+---+---+
```

2. 用两个hash函数对d1进行hash，假设hash结果跟bit数组的长度取余之后为分别是2和8， ，那么将bit数组的第2和第8位设置位1
```
  0   1   2   3   4   5   6   7   8   9
+---+---+---+---+---+---+---+---+---+---+
| 0 | 0 | 1 | 0 | 0 | 1 | 0 | 0 | 1 | 0 |
+---+---+---+---+---+---+---+---+---+---+
```

### 判断是否存在

当需要读取data set的数据的时候，先访问bloomfilter，判断数据是否存在，如果不存在，那么就可以提前直接返回，不需要再访问data set了。

判断是否存在和上面的插入操作一样，假设现在访问d1，那么同样的用两个hash函数对d1进行hash，然后判断第2和第5位是不是都是1，如果都是1，表示d1有可能存在于data set中，需要继续访问data set，只要有任何一位不是1，表示一定不存在，可以直接返回。这里为什么是有可能存在呢？因为bloomfilter不支持update和remove操作，所以如果d1已经从data set中移除了，bloomfilter是不知道的。

为什么bloomfilter不支持update和remove呢？假设现在有一个d3，hash后的位置是5和6，于是bit数组变成了下面这样：
```
  0   1   2   3   4   5   6   7   8   9
+---+---+---+---+---+---+---+---+---+---+
| 0 | 0 | 1 | 0 | 0 | 1 | 1 | 0 | 1 | 0 |
+---+---+---+---+---+---+---+---+---+---+
```

接着删除掉d2和d3，如果只是简单的将d2的2和8以及d3的5和6位设置为0的话，整个bit数组就会变成全0，这样在读取d1的时候，就会报d1不存在，但实际上d1还在data set里面，于是出现了正确性的问题。

## 高级

### False Positive

表示bloomfilter报数据存在，但在data set中找不到数据的情况，一般是由下面这些原因引起的

1. data set的数据被更新或者删除
2. 有hash结果一样的数据（由于是多个hash函数，这种情况几率较小）
3. bit数组比较满了，导致大家的hash结果相互重叠。（对于上面的bloomfilter，如果来读取一个d4，假设它的hash结果是6和8，那么bloomfilter就会报数据存在，可实际上data set中没有d4，于是出现False Positive.）

### FPP(False Positive Probability)

表示bloomfilter报数据存在，但在data set中找不到数据的几率，这个数字越小越好，一般由data set的大小，bit数组的长度以及hash函数的个数决定。有一个通用的公式计算预期的FPP=(1-e^(-kn/m))^k，这里e是常数，n是预估的data set的大小，m是bit数组的长度，k是hash函数的个数。
公式详情请参考[wikipedia关于Bloom filter的介绍](https://en.wikipedia.org/wiki/Bloom_filter)

### bit数组

理想情况下我们希望的是bit数组越小越好，这样能节约内存，但太小了之后会导致FPP偏高。

### hash函数

对于hash函数而言，最好是结果足够分散，并且速度快，hash函数之间相互独立。hash函数越多，运算速度越慢，bit数组越容易被充满，bit数组越满FPP就会越高，但如果hash函数太少，会导致冲突变多，也会影响FPP。它的理想值k = (m/n)ln(2)，n是预估的data set的大小，m是bit数组的长度

### 初始化bloomfilter

有了上面这些影响bloomfilter的参数，我们就得到了初始化bloomfilter的过程：
1. 评估data set的大小n
2. 根据可供支配内存的大小选择一个bit数组的长度m
3. 根据前两步的m和n, 计算最有的hash函数个数k = (m/n)ln(2)
4. 再根据上面FPP里面介绍的公式，由前三步的n,m,k计算出预期的FPP，如果FPP值高于自己的要求，那么重新回到第二步，增加数组的长度m，接着重复3和4步，直到FPP值达到预期。

## 注意
1. 如果实际的data set大小比预估的大，那么得到的实际FPP也会高出预期值。
2. 如果update和remove操作的比例比较高，那么实际的FPP也会高出预期

所以对于bloomfilter来说，预估一定要尽量准确，大了浪费空间，小了效果会打折，同时update和remove所占的比例不能太高，否则效果同样会打折。在实际使用过程中，一定要用实际的数据实测一下，看具体效果如何，并根据测试结果调整m和期待的FPP大小。

## 参考

* [Bloom filter](https://en.wikipedia.org/wiki/Bloom_filter)
* [bloomfilter tutorial](http://llimllib.github.io/bloomfilter-tutorial/)