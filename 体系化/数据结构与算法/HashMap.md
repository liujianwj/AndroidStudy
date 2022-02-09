## HashMap

​      大家都知道**ArrayList**的特点是查找快、修改快，删除和添加慢，而**LinkedList**插入和删除快，但是查找、修改耗时，那有没有一种数据结构能能同时满足这两种需求呢？这个时候就需要引入今天主角----HaspMap。

#### HaspMap 数据结构

在JDK1.7之前，HashMap的数据结构是：数组+链表

在JDK1.7之后，HashMap的数据结构是：数组+链表+红黑树

存储方式key->value的方式，那么key是怎么对应数组中的下标呢？引入hash算法，首先对key做hash处理，得到int类型的hashCode，然后用hashCode和数组长度求膜，就得到数组长度范围内的index。

装箱put过程：

1、Key(Object) -> hashCode(int) : 将Key做hash处理

2、index(0~length-1  ) = hashCode % length -> (hashCode & (length - 1))：为了获得数组范围内下标而做求膜，根据CPU位运算优化，优化位与运算处理，最终得到index。

index下标处，通过链表的方式往后存储，这样就会有个问题，当hasp频繁的时候，就类似于单链表的存储和读取了。

![这里写图片描述](https://img-blog.csdn.net/20150820130200565)

#### 结构优化

为了解决上述冲突问题，HashMap中引入以下几个点：

- 扩容
- 加载因子
- 阈值
- 扰动函数 https://www.zhihu.com/question/20733617

​     HaspMap的数组初始大小是16，而加载因子大小为0.75，则此时的阈值=16 * 0.75 = 12，也就是当数组中存储的数量大于12时，会对数组进行扩容处理，扩容的方式是length = length * 2，扩容之后碰撞的概率就会变小，但是由于扩容的时候，需要对原先的数据都重新做hasp处理获取新的index并存储，这里就消耗了很大的性能。

​    会发现length是2的次幂，为什么呢？主要是跟index的求值公式 hashCode & (length - 1 )有关，我们可以看下这个例子：

 **当 lenth = 2n 时，X % length = X & (length - 1)**，用位移替代取模运算，提升性能。

如果只是这样的话，那hashCode 每次只能用到低位数值，碰撞概率高，这个时候可以用扰动函数

```java
/**
* JDK 8 的 hash 方法
*/
static final int hash(Object key) {
int h;
return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
}
```

通过右移16位，然后进行异或操作，这样高位数也变现参与进来，碰撞概率低。

假设当前有hashCode1 = 6-> 110，hashCode2 = 7-> 111，

当 length=16时，length-1=16-1=15->1111,

则：  hashCode & (length - 1 ) ：110 & 1111 = 110 = 6，111 & 1111 = 111 = 7；

当length =17时， length-1=17-1=16->10000,

则：  hashCode & (length - 1 ) ：110 & 10000 = 0 ，111 & 10000 = 0 ；

当length =10时， length-1=10-1=9->1001,

则：  hashCode & (length - 1 ) ：110 & 1001 = 0 ，111 & 1001 = 0001=1；

会发现当length为2的次幂时，length - 1对应的位数上都是1，则有效位数越多，hashCode & (length - 1 )相与后的结果可能性越多，碰撞的概率越低。



#### HaspMap缺点

内存浪费，至少25%，空间换时间的处理，尤其在扩容的时候，需要大量的处理以及数组的copy

#### 优化替换

**SparseArray：**采用双数组的方式，一个数组存储key，一个存储按key的顺序记录的value，key必须为int类型。

![img](https://5b0988e595225.cdn.sohucs.com/images/20190516/75e7c8a963894362902d2f0a7e7ad4bb.png)

增加：根据key值进行二分查找，找到可以添加元素的位置，然后插入数据，并移动其他数据

查找：根据key值进行二分查找，找到数组下标，取出对应value值

删除：先根据key值找到数组位置，再从链表中删除节点，删除只是标记为delete，减少数组移动操作，且下次可以直接复用

为了解决SparseArray中key必须为int类型的缺点，引入ArrayMap

**ArrayMap：**HaspMap + Sparse

![这里写图片描述](https://img-blog.csdn.net/20150922094132100)

原理：其内部也是采用了两个数组，一个数组存储对key(Object)做hash过后的值，另一个数组按key的顺序记录的key-value值，冲突时采用追加的方式解决，比如说index=20冲突，则存储在index=21处，查询的的时候也是根据二分查找，由于冲突时采用追加方式，所以还需要按index++对比key值。由于采用数组的方式，所以对插入和删除效率比较低。

