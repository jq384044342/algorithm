# Redis

**前言**



我们知道redis支持五种数据类型，其实这五种类型就是五种数据对象。我们不曾注意到。其实，实际工作中，我们操作的每一个命令，底层都至少会创建两个对象。一个是键对象、一个是值对象。



今天我们就来学习一下，redis中的五种数据对象是如何工作的，它们又有哪些特性呢？



![图片](https://mmbiz.qpic.cn/mmbiz_jpg/Z33JZWhn1uNlqRNKxe0QFs8SmaxHGktTFcIxica7fra65Q5kuWVUt8gZASxgcwWrbibtDhEKM0owJsH04ZpHyEzA/640?wx_fmt=jpeg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)



**对象的属性**

redis 的键和值都是一个对象，每个对象都有以下五个属性：类型、编码、指针、引用计数、空转时长。

```c
typedef struct redisObject {    
    unsigned type: 4; #类型    
    unsigned encoding:4; #编码    
    int refcount; #引用计数  
    unsigned lru:22; #空转时长    
    void *ptr; #指向底层实现数据结构的指针  
}
```



**type** 属性，可为以下五种的其中一种：字符串、列表、哈希、集合、有序集合

**refcount** 属性，用于记录该对象被引用的次数，当引用计数为0时，对象会释放

**lru** 属性，用于记录对象最后一次访问的时间，若访问的时间过久，对象会释放

**ptr** 属性，用于指向对象的底层实现的数据结构，而数据结构是由encoding决定的

**encoding** 属性，记录了对象所使用的编码，也就是说，对象底层使用了哪种数据结构作为对象的底层实现。属性值可以是以下表格中的一个。



![图片](https://mmbiz.qpic.cn/mmbiz_jpg/Z33JZWhn1uPgQL8CSDtqg7msF5BJmsnPciavHyKuvnQ7lO0rGiavgaBQYPiaOibZlPgQfTCmqOAI3ib5fW23r8BtUmg/640?wx_fmt=jpeg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)



![图片](https://mmbiz.qpic.cn/mmbiz_png/4o22OFcmzHkibCHbibcjuQ2RR5ITRsxibiabctV3kTbcOwnfuibuH1Qbial40SKT6ppxDjIKibcCyWHP9iaZpD437uzcBQ/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)



**TYPE** 命令，可以查看一个数据库键的值对象的类型

**OBJECT ENCODING** 命令，可以查看一个数据库键的值对象的编码

```
127.0.0.1:6379> set msg 'hello word'
OK
127.0.0.1:6379> get msg
"hello word"
127.0.0.1:6379> TYPE msg
string
127.0.0.1:6379> OBJECT ENCODING msg
"embstr"
127.0.0.1:6379>
```



注意：每种类型的对象都至少使用了两种以上不同的编码方式。通过改变 encoding 的方式，来提高 redis 的灵活性和效率，因为 redis 会根据不同的场景，来为对象设置不同的编码，从而优化对象在某一场景下的效率。





**一、字符串对象**



字符串的编码可以是以下三种：int、raw、embstr；



**int** 编码，redis存储的是一个整数值，该整数值可以用long类型表示时，使用int编码。

**raw** 编码，redis存储的是一个字符串，该字符串长度大于39个字节时，使用raw编码。

**embstr** 编码，redis存储的是一个字符串，且长度大于等于39个字节时，使用embstr编码。



**raw 和 embstr 编码的区别：**

raw 会调用两次内存分配机制，内存不是连续空间。释放内存时也需要调用两次函数。

embstr 会调用一次内存分配机制，内存是连续的空间。释放内存只需要调用一次函数。



**1.1 编码的转换**



因为append命令只允许对 raw 编码的字符串对象进行操作。当我们对一个 int 编码的 或 embstr 编码的字符串对象进行一定的操作时，会将编码转换为 raw 编码的字符串对象。





```
127.0.0.1:6379> set msg 123
OK
127.0.0.1:6379> OBJECT ENCODING msg
"int"
127.0.0.1:6379> append msg 222
(integer) 6
127.0.0.1:6379> OBJECT ENCODING msg
"raw"
127.0.0.1:6379>
```

```
127.0.0.1:6379> set a 'abc'OK127.0.0.1:6379> object encoding a"embstr"127.0.0.1:6379> append a 123(integer) 6127.0.0.1:6379> object encoding a"raw"127.0.0.1:6379>
```



以上命令，则是实践 int 和 embstr 编码，通过 append 转换为 raw 编码类型的过程。





**二、列表对象**



早期的列表对象的编码是由 ziplist 、 linkedlist，也就是说当元素少时使用ziplist，当元素多时使用 linkedlist。但是后来，新版本中对列表数据结构进行了改造，使用 **quicklist** 代替了 ziplist 和 linkedlist；



**quicklist 编码，**是使用快速列表作为列表对象的底层实现。

**ziplist 编码**，是使用压缩列表作为列表对象的底层实现。

**linkedlist 编码**，是使用双端链表作为列表对象的底层实现。



下面命令，体现出新版本中使用“快速列表”作为底层数据结构的实现



```
127.0.0.1:6379> lpush list 1
(integer) 1
127.0.0.1:6379> object encoding list
"quicklist"
127.0.0.1:6379>
```



quicklist 是 ziplist 和 linkedlist 的混合体，它将 linkedlist 按段切分，每一段使用 ziplist 存储，多个 ziplist 之间使用双向指针串接起来。如下图所示。



![图片](https://mmbiz.qpic.cn/mmbiz_jpg/Z33JZWhn1uPgQL8CSDtqg7msF5BJmsnPTSIwe38L1u4RZMe6jVIwPMwqabeWAZhUDYF5ynpaSVQCNzfsrnQ7aQ/640?wx_fmt=jpeg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)



- quicklist 内部默认单个 ziplist 长度为 8k 字节，超出了这个字节数，就会新起一个 ziplist。
- ziplist 的长度由配置参数 list-max-ziplist-size 决定。
- 快速列表 默认的压缩深度是0，也就是不压缩。压缩的实际深度可由list-compress-depth 决定。
- 为了支持快速的 push/pop 操作，quicklist 的首尾两个 ziplist 不压缩，此时深度就是 1。
- 如果深度为 2，就表示 quicklist 的首尾第一个 ziplist 以及首尾第二个 ziplist 都不压缩。



**2.1 快速列表的数据结构、节点**





```c
#快速列表
struct quicklist {    
quicklistNode* head;    
quicklistNode* tail;    
long count; #元素总数    
int nodes; #ziplist 节点的个数    
int compressDepth; # LZF 算法压缩深度
}
```



```c
#快速列表节点
struct quicklistNode {   
quicklistNode* prev;    
quicklistNode* next;    
ziplist* zl; #指向压缩列表    
int32 size; #ziplist 的字节总数    
int16 count; #ziplist 中的元素数量    
int2 encoding; #存储形式 2bit，原生字节数组还是 LZF 压缩存储
}
```



**2.2 双端链表、压缩列表、\**快速列表\**的特点**

双端链表，进行push、pop来说速度很快，但是它的内存开销比较大，因为它要额外存储前驱指针和后继指针。链表采用不连续的内存空间，所以它对修改和插入操作来说相当快速，但是容易产生内存碎片。



压缩列表，仅限于存储字节较短的数据，因为它是为节省内存而开发的数据结构，压缩列表使用连续的内存空间，所以它对删除和插入的效率相对较低，但是查询效率很高。压缩列表对于插入很大的数据情况下，会产生扩容和大量拷贝流程，此时效率相对较低。



快速列表是对压缩列表和双端链表的空间和时间效率的折中产物。它结合了二者的优点。





**三、哈希对象**



哈希对象的编码方式有 ziplist 、hashtable



**ziplist** 编码，哈希对象底层使用压缩列表作为底层实现。

**hashtable** 编码，哈希对象底层使用字典作为底层实现。

下面命令，是压缩列表实现的哈希对象存储。



```
127.0.0.1:6379> hset mytest name "cici"
(integer) 1
127.0.0.1:6379> hset mytest age 18
(integer) 1
127.0.0.1:6379> hset mytest sex '女'
(integer) 1
127.0.0.1:6379> object encoding mytest
"ziplist"
127.0.0.1:6379>
```



![图片](https://mmbiz.qpic.cn/mmbiz_jpg/Z33JZWhn1uPgQL8CSDtqg7msF5BJmsnPTsvbaRS8R1TicnmKyzRGVQcTyXBuwqicXxG5ickAUsVaC2Gk6cQXoxrTA/640?wx_fmt=jpeg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)



下面命令，是字典实现的哈希对象存储。



```
127.0.0.1:6379> hset mytest content '我是中国人，我在测试哈希，你看到了吗？我是中国人，我在测试哈希，你看到了吗？我是中国人，我在测试哈希，你看到了吗？我是中国人，我在测试哈希，你看到了吗？我是中国人，我在测试哈希，你看到了吗？我是中国人，我在测试哈希，你看到了吗？我是中国人，我在测试哈希，你看到了吗？'
127.0.0.1:6379> object encoding mytest
"hashtable"
127.0.0.1:6379>
```



当 ziplist 作为哈希对象的底层实现时，若再次存储键值，长度大于等于64字节时，哈希对象的底层实现，会进行编码转换，此时的哈希对象的编码方式为 hashtable 。



![图片](https://mmbiz.qpic.cn/mmbiz_jpg/Z33JZWhn1uPgQL8CSDtqg7msF5BJmsnPD3WLvaVybjhWpdM0M6bsx59XnTiaIPYLfkcicUC8EQ0DLicV32v72aHQw/640?wx_fmt=jpeg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)



如上图所示，hashtable 编码的哈希对象，使用字典作为底层实现，哈希对象中的每个键值对都使用一个字典键值对来存储。字典的每个键和值都是一个字符串对象，对象中分别存存了键值对的键和值。



**3.1 编码转换**



当哈希对象同时满足以下两个条件时，默认情况下哈希对象会使用压缩 ziplist 编码；



- 哈希对象保存的所有键值对的键和值长度小于64字节
- 哈希对象保存的键值对数量小于 512 个



以上两个条件可以通过配置文件进行修改。若不满足以上两个条件的情况下，将会采用 hashtable 进行编码。



**四、集合对象**



集合对象的编码有 intset、hashtable



**intset** 编码，集合对象使用整数集合作为底层实现，集合元素都存储在整数集合里。

**hashtable** 编码，集合对象使用字典作为底层实现。

下面命令，是整数集合实现的集合对象存储。



```
127.0.0.1:6379> sadd mynumber 111 222 333
(integer) 3
127.0.0.1:6379> object encoding mynumber
"intset"
127.0.0.1:6379>
```



![图片](https://mmbiz.qpic.cn/mmbiz_jpg/Z33JZWhn1uPgQL8CSDtqg7msF5BJmsnP6FdjDofEzdSabFs0mVXnXiaYEnuFH6pLzvpoOc16v3rjVUUJUiaEHIxg/640?wx_fmt=jpeg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)



下面命令，是字典实现的集合对象存储



```
127.0.0.1:6379> sadd mynumber "hello" "你好"
(integer) 2
127.0.0.1:6379> object encoding mynumber
"hashtable"
127.0.0.1:6379>
```



下面命令，是查看集合中的元素数量 和 元素值，由此可见，集合编码发生了变化，已经从 intset 编码转换成 hashtable 编码。



```
127.0.0.1:6379> scard mynumber
(integer) 5
127.0.0.1:6379> smembers mynumber
1) "hello"
2) "333"
3) "111"
4) "222"
5) "\xe4\xbd\xa0\xe5\xa5\xbd"
127.0.0.1:6379>
```



![图片](https://mmbiz.qpic.cn/mmbiz_jpg/Z33JZWhn1uPgQL8CSDtqg7msF5BJmsnPbwPRpo8Z3iaMfGD4Ly4yERO7bvHM1Zuspb8rK6Un2yVD5KMIn4JoKjQ/640?wx_fmt=jpeg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)



由上图可见，集合中因为添加了字符串类型值，因此从原本的 intset 编码转为 hashtable 编码。



**4.1 编码转换**



当集合对象同时满足以下两个条件时，对象会使用 intset 编码



- 集合对象保存的所有元素都是整数值。
- 集合对象保存的元素数量不超过 512 个。



以上两个条件是可以修改的，通过修改配置文件即可。若不满足以上条件的集合对象将使用 hashtable 编码。





**五、有序集合对象**



有序集合对象是有 ziplist 、skiplist



**ziplist** 编码，有序集合对象使用压缩列表作为底层实现。

**skiplist** 编码，有序集合对象使用 zset 结构作为底层实现，一个zset包含一个字典和一个跳跃表。



下面命令，是压缩列表作为有序集合对象的底层实现。



```
127.0.0.1:6379> zadd price 5.6 apple 3.4 orange
(integer) 2
127.0.0.1:6379> object encoding price
"ziplist"
127.0.0.1:6379>
```



![图片](https://mmbiz.qpic.cn/mmbiz_jpg/Z33JZWhn1uPgQL8CSDtqg7msF5BJmsnPWeR8ajxgibBt9XgRriaLiaia6JlAn1aZyhoW1oN3D1uwbpDUpM3kSbR01A/640?wx_fmt=jpeg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)



注意：有序集合在压缩列表中按分值大小进行排序存储。

下面命令，是跳跃表作为有序集合对象的底层实现，它是一个 zset 结构。



```
127.0.0.1:6379> zadd price 4.4 "买了一件衣服，这个衣服真好看，真的很好看，真的很好看，真的很好看"(integer) 1
127.0.0.1:6379> object encoding price
"skiplist"
127.0.0.1:6379>
```



下面命令，是查看有序集合的元素个数和排序情况。

```
127.0.0.1:6379> zcard price
(integer) 3
127.0.0.1:6379> ZRANGE price 0 -1
1) "orange"
2) "\xe4\xb9\xb0\xe4\xba\x86\xe4\xb8\x80\xe4\xbb\xb6\xe8\xa1\xa3\xe6\x9c\x8d\xef\xbc\x8c\xe8\xbf\x99\xe4\xb8\xaa\xe8\xa1\xa3\xe6\x9c\x8d\xe7\x9c\x9f\xe5\xa5\xbd\xe7\x9c\x8b\xef\xbc\x8c\xe7\x9c\x9f\xe7\x9a\x84\xe5\xbe\x88\xe5\xa5\xbd\xe7\x9c\x8b\xef\xbc\x8c\xe7\x9c\x9f\xe7\x9a\x84\xe5\xbe\x88\xe5\xa5\xbd\xe7\x9c\x8b\xef\xbc\x8c\xe7\x9c\x9f\xe7\x9a\x84\xe5\xbe\x88\xe5\xa5\xbd\xe7\x9c\x8b"3) "apple"
127.0.0.1:6379>
```



![图片](https://mmbiz.qpic.cn/mmbiz_jpg/Z33JZWhn1uPgQL8CSDtqg7msF5BJmsnPcTkXEa4bdgGU1VATicOAhHfXhxOQ2oPaa3WruZPpTbxRicO5snKK9G6g/640?wx_fmt=jpeg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)



由上图可见，当有序集合使用了 skiplist 编码方式，其实底层采用了zset结构来存储了数据内容。



zset 结构分别用了一个字典 和 一个跳跃表来完成底层实现。

字典的优点，在于它以时间复杂度为O(1) 的速度取值。

跳跃表的优点，在于它以分值进行从小到大的排序。结合二者的优点作为 zset 的整体结构来完成了有序集合的底层实现。



**5.1 编码的转换**



有序集合对象同时满足以下两个条件时，对象使用ziplist 编码



- 有序集合保存的元素数量小于128个。
- 有序集合保存的所有元素成员长度都小于64字节。



以上两个条件是可以修改的，通过修改配置文件即可。若不满足以上条件的有序集合对象将使用 skiplist 编码。



Sorted Set的数据结构是一种跳表，即SkipList，如下图所示，红线是查找10的过程：

![图片](https://mmbiz.qpic.cn/mmbiz_png/4o22OFcmzHkibCHbibcjuQ2RR5ITRsxibiabQCmG3icsOpxibHMmAVFTTCufywXTCh76DqgpoCGHLBdZahyCibT4K0mlg/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)SkipList

- 如何借助Sorted set实现多维排序

Sorted  Set默认情况下只能根据一个因子score进行排序。如此一来，局限性就很大，举个栗子：热门排行榜需要按照下载量&最近更新时间排序，即类似数据库中的ORDER BY download_count, update_time DESC。那这样的需求如果用Redis的Sorted Set实现呢？

事实上很简单，思路就是将涉及排序的多个维度的列通过一定的方式转换成一个特殊的列，即result = function(x, y,  z)，即x，y，z是三个排序因子，例如下载量、时间等，通过自定义函数function()计算得到result，将result作为Sorted  Set中的score的值，就能实现任意维度的排序需求了。可以参考笔者之前的文章：《[Redis高级玩法：如何利用SortedSet实现多维度排序](http://mp.weixin.qq.com/s?__biz=MzU5ODUwNzY1Nw==&mid=2247484672&idx=1&sn=34b75672e83e403429504eed420ed299&chksm=fe426ce6c935e5f00e2f3819e9e59caa29b596846bfcc722d0c30be40e5d338471c7beefb915&scene=21#wechat_redirect)》。





**六、多态命令的实现**



前面我们说过，对象包含类型、编码、指针、引用计数、空转时长 五个属性。编码在和指针在前面已经分别阐述了具体的实现方式。那类型主要用于哪方面呢？



其实，类型主要用于判断用户执行一个命令时，检测命令的键是否能够执行该命令。如果可以执行，则服务器就对键执行指定的命令，否则，服务器会拒绝执行命令，并抛出一个异常。



多态主要的体现，是当一个对象的底层实现，不止一种编码方式时，需要根据编码方式进行区分，来选择正确的命令执行方式。



![图片](https://mmbiz.qpic.cn/mmbiz_jpg/Z33JZWhn1uNlqRNKxe0QFs8SmaxHGktTdziaLI3RABFBDezQUOcj0s8eI1P9hsnrmPVthWH2O6CaWROsABEDxZQ/640?wx_fmt=jpeg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)





**七、引用计数、空转时长**



1）Redis在自己的对象系统中，构建了一个对象引用计数，可用于内存回收机制，也可实现对象的共享机制，让多个数据库共享一个对象，用于节约内存的作用。

2）对象带有访问时间记录信息，用于删除空间转时间较大的那个键。

# 3种高级数据结构

Redis中3种高级数据结构分别是bitmap、GEO、HyperLogLog，针对这3种数据结构，笔者之前也有文章介绍过。其中，最重要的就是**bitmap**。

### bitmap

这个就是Redis实现的BloomFilter，BloomFilter非常简单，如下图所示，假设已经有3个元素a、b和c，分别通过3个hash算法h1()、h2()和h2()计算然后对一个bit进行赋值，接下来假设需要判断d是否已经存在，那么也需要使用3个hash算法h1()、h2()和h2()对d进行计算，然后得到3个bit的值，恰好这3个bit的值为1，这就能够说明：**d可能存在集合中**。再判断e，由于h1(e)算出来的bit之前的值是0，那么说明：**e一定不存在集合中**：

![图片](https://mmbiz.qpic.cn/mmbiz_png/4o22OFcmzHkibCHbibcjuQ2RR5ITRsxibiabvlYodLib40LkOOt8SOmBKKFwWQuZUnVibMOJPQwg2CvQnJkqJWqiaiahbA/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)BloomFilter

需要说明的是，bitmap并不是一种真实的数据结构，它本质上是String数据结构，只不过操作的粒度变成了位，即bit。因为String类型最大长度为512MB，所以bitmap最多可以存储2^32个bit。

### GEO

GEO数据结构可以在Redis中存储地理坐标，并且坐标有限制，由EPSG:900913 / EPSG:3785 / OSGEO:41001 规定如下：

1. 有效的经度从-180度到180度。
2. 有效的纬度从-85.05112878度到85.05112878度。

当坐标位置超出上述指定范围时，该命令将会返回一个错误。添加地理位置命令如下：

```
redis> GEOADD city 114.031040 22.324386 "shenzhen" 112.572154 22.267832 "guangzhou"
(integer) 2
redis> GEODIST city shenzhen guangzhou
"150265.8106"
```

但是，需要说明的是，Geo本身不是一种数据结构，它**本质上还是借助于Sorted Set（ZSET）**，并且使用**GeoHash**技术进行填充。Redis中将经纬度使用52位的整数进行编码，放进zset中，score就是GeoHash的52位整数值。在使用Redis进行Geo查询时，其内部对应的操作其实就是zset(skiplist)的操作。通过zset的score进行排序就可以得到坐标附近的其它元素，通过将score还原成坐标值就可以得到元素的原始坐标。

总之，Redis中处理这些地理位置坐标点的思想是：二维平面坐标点 --> 一维整数编码值 --> zset(score为编码值) -->  zrangebyrank(获取score相近的元素)、zrangebyscore --> 通过score(整数编码值)反解坐标点  --> 附近点的地理位置坐标。

- GEOHASH原理

使用wiki上的例子，纬度为42.6，经度为-5.6的点，转化为base32的话要如何转呢？
首先拿纬度来进行说明，纬度的范围为-90到90，将这个范围划为两段，则为[-90,0]、[0,90]，然后看给定的纬度在哪个范围，在前面的范围的话，就设当前位为0，后面的话值便为1.然后继续将确定的范围1分为2，继续以确定值在前段还是后段来确定bit的值。就这样慢慢的缩小范围，一般最多缩小13次就可以了(经纬度的二进制位相加最多25位，经度13位，纬度12位)。这时的中间值，将跟给定的值最相近。如下图所示：

![图片](https://mmbiz.qpic.cn/mmbiz_png/4o22OFcmzHkibCHbibcjuQ2RR5ITRsxibiabiaDSknNiaVNOyalgNhJjMkNd5KjnuJlUa1olsHnUicvoaeIg4lfs3GnqQ/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)Geohash

第1行，纬度42.6位于[0, 90]之间，所以bit=1；第2行，纬度42.6位于[0, 45]之间，所以bit=0；第3行，纬度42.6位于[22.5,  45]之间，所以bit=1，以此类推。这样，取出图中的bit位：1011 1100 1001，同样的方法，将经度(范围-180到180)算出来为 ：0111 1100 0000 0。结果对其如下：

```
# 经度
0111 1100 0000 0
# 纬度
1011 1100 1001
```

得到了经纬度的二进制位后，下面需要将两者进行结合：从经度、纬度的循环，每次取其二进制的一位(不足位取0)，合并为新的二进制数：01101111 11110000 01000001  0。每5位为一个十进制数，结合base32对应表映射为base32值为：ezs42。这样就完成了encode的过程。

# Streams

这是Redis5.0引入的全新数据结构，这种数据结构笔者之前也有文章对其进行详细解读，链接地址：《[Streams：深入剖析Redis5.0全新数据结构](http://mp.weixin.qq.com/s?__biz=MzU5ODUwNzY1Nw==&mid=2247484059&idx=1&sn=83da3a7e3ec3219e0792129e27e9f7bc&chksm=fe426b7dc935e26b4c05c4e4755bc3308a32799a3ac8ed4521040522dd678258ded12101ccf2&scene=21#wechat_redirect)》，用一句话概括Streams就是Redis实现的内存版kafka。而且，Streams也有**Consumer Groups**的概念。通过Redis源码中对stream的定义我们可知，streams底层的数据结构是**radix tree**：

```
typedef struct stream {
    rax *rax;               /* The radix tree holding the stream. */
    uint64_t length;        /* Number of elements inside this stream. */
    streamID last_id;       /* Zero if there are yet no items. */
    rax *cgroups;           /* Consumer groups dictionary: name -> streamCG */
} stream;
```

那么这个radix tree长啥样呢？在Redis源码的rax.h文件中有一段这样的描述，这样看起来是不是就比较直观了：

```
 *                    (f) ""
 *                    /
 *                 (i o) "f"
 *                 /   \
 *    "firs"  ("rst")  (o) "fo"
 *              /        \
 *    "first" []       [t   b] "foo"
 *                     /     \
 *           "foot" ("er")    ("ar") "foob"
 *                    /          \
 *          "footer" []          [] "foobar"
```

**Radix Tree(基数树) 事实上就几乎相同是传统的二叉树**。仅仅是在寻找方式上，以一个unsigned  int类型数为例，利用这个数的每个比特位作为树节点的推断。能够这样说，比方一个数10001010101010110101010，那么依照Radix  树的插入就是在根节点，假设遇到0，就指向左节点，假设遇到1就指向右节点，在插入过程中构造树节点，在删除过程中删除树节点。如下是一个保存了7个单词的Radix Tree：

![图片](https://mmbiz.qpic.cn/mmbiz_png/4o22OFcmzHkibCHbibcjuQ2RR5ITRsxibiabeQJxnxp80LNBWEnxShCwHwOdD83L4ib2AycJQDIia7kyEacgmpibuDibyQ/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)radix tree





**八、重点回顾**



本文主要学习到，redis的五大数据对象，我们知道redis中的每个键值对其实都是一个对象，而每个对象的底层实现都至少采用两种以上的编码方式来提高redis的性能和效率。另外，redis还支持引用计数和内存共享机制。用于提高对象的最大化利用和快速释放无用对象。当我们执行每一个redis命令时，redis都会首先检测该命令的键是否支持执行该命令，若不支持，则报类型异常。

### 1.string

1. **介绍：** string 是 redis 最基本的类型，可以理解成与 memcached 一模一样的类型，一个 key 对应一个 value。value  不仅是 string，也可以是数字。string 类型是二进制安全的，意思是 redis 的 string 类型可以包含任何数据，比如 jpg  图片或者序列化的对象。string 类型的值最大能存储 512M。
2. **常用命令:** set,get,decr,incr,mget 等。
3. **应用场景** ：常规 key-value 缓存应用；常规计数：微博数，粉丝数等。

### 2.Hash

1. **介绍** ：Hash 是一个键值（key-value）的集合。redis 的 hash 是一个 string 的 key 和 value 的映射表，Hash 特别适合存储对象。常用命令：hget,hset,hgetall 等。
2. **常用命令** ：hget,hset,hgetall 等。
3. **应用场景** ：hash 特别适合用于存储对象，后续操作的时候，你可以直接仅仅修改这个对象中的某个字段的值。比如我们可以 hash 数据结构来存储用户信息，商品信息等等。比如下面我就用 hash 类型存放了我本人的一些信息：

```
key=JavaUser293847
value={
  “id”: 1,
  “name”: “SnailClimb”,
  “age”: 24,
  “location”: “Wuhan, Hubei”
}
```

### 3.list

1. **介绍** ：list 列表是简单的字符串列表，按照插入顺序排序。可以添加一个元素到列表的头部（左边）或者尾部（右边） ：
2. **常用命令：**lpush、rpush、lpop、rpop、lrange(获取列表片段)等。
3. **应用场景** ：list 应用场景非常多，也是 Redis 最重要的数据结构之一，比如 twitter 的关注列表，粉丝列表都可以用 list 结构来实现。

> **list 就是链表，可以用来当消息队列用。redis 提供了 List 的 push 和 pop 操作，还提供了操作某一段的  api，可以直接查询或者删除某一段的元素。redis list  的是实现是一个双向链表，既可以支持反向查找和遍历，更方便操作，不过带来了额外的内存开销。**

### 4.set

1. **介绍** ：set 是 string 类型的无序集合。集合是通过 hashtable 实现的。set 中的元素是没有顺序的，而且是没有重复的。
2. **常用命令：** sdd、spop、smembers、sunion 等。
3. **应用场景** ：redis set 对外提供的功能和 list 一样是一个列表，特殊之处在于 set 是自动去重的，而且 set 提供了判断某个成员是否在一个 set 集合中。

### 5.zset

1. **介绍** ：zset 和 set 一样是 string 类型元素的集合，且不允许重复的元素。
2. **常用命令：** zadd、zrange、zrem、zcard 等。
3. **使用场景：**sorted set  可以通过用户额外提供一个优先级（score）的参数来为成员排序，并且是插入有序的，即自动排序。当你需要一个有序的并且不重复的集合列表，那么可以选择 sorted set 结构。和 set 相比，sorted set 关联了一个 double 类型权重的参数  score，使得集合中的元素能够按照 score 进行有序排列，redis 正是通过分数来为集合中的成员进行从小到大的排序。

> **Redis sorted set 的内部使用 HashMap 和跳跃表(skipList)来保证数据的存储和有序，HashMap 里放的是成员到  score 的映射，而跳跃表里存放的是所有的成员，排序依据是 HashMap 里存的  score，使用跳跃表的结构可以获得比较高的查找效率，并且在实现上比较简单。**

**Redis 数据类型应用场景总结:**

![图片](https://mmbiz.qpic.cn/mmbiz_png/iaIdQfEric9TzlTcnXg26t1Dia266foajMicLwYfVaFWKQapY9hjXAaT8J5Oasmt7qEiadkZJFNLucBF04YeO597Q1A/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)



**我** ：我是结合 Spring Boot 使用的。一般有两种方式：

1. 接通过 `RedisTemplate` 来使用
2. 使用 Spring Cache 集成 Redis pom.xml 中加入以下依赖：

简单说一下代码吧！

```xml
<dependencies>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-data-redis</artifactId>
    </dependency>
    <dependency>
        <groupId>org.apache.commons</groupId>
        <artifactId>commons-pool2</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.session</groupId>
        <artifactId>spring-session-data-redis</artifactId>
    </dependency>

    <dependency>
        <groupId>org.projectlombok</groupId>
        <artifactId>lombok</artifactId>
        <optional>true</optional>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-test</artifactId>
        <scope>test</scope>
    </dependency>
</dependencies>
```

- **spring-boot-starter-data-redis** :在 spring boot 2.x 以后底层不再使用 Jedis，而是换成了 Lettuce。
- **commons-pool2** ：用作 redis 连接池，如不引入启动会报错
- **spring-session-data-redis** ：spring session 引入，用作共享 session。配置文件

`application.yml` 的配置：

```yml
server:
  port: 8082
  servlet:
    session:
      timeout: 30ms
spring:
  cache:
    type: redis
  redis:
    host: 127.0.0.1
    port: 6379
    password:
    # redis默认情况下有16个分片，这里配置具体使用的分片，默认为0
    database: 0
    lettuce:
      pool:
        # 连接池最大连接数(使用负数表示没有限制),默认8
        max-active: 100
```

创建实体类 `User.java`

```java
public class User implements Serializable{

    private static final long serialVersionUID = 662692455422902539L;

    private Integer id;

    private String name;

    private Integer age;

    public User() {
    }

    public User(Integer id, String name, Integer age) {
        this.id = id;
        this.name = name;
        this.age = age;
    }

    public Integer getId() {
        return id;
    }

    public void setId(Integer id) {
        this.id = id;
    }

    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }

    public Integer getAge() {
        return age;
    }

    public void setAge(Integer age) {
        this.age = age;
    }

    @Override
    public String toString() {
        return "User{" +
                "id=" + id +
                ", name='" + name + '\'' +
                ", age=" + age +
                '}';
    }
}
```

### RedisTemplate 的使用方式

默认情况下的模板只能支持 `RedisTemplate<String, String>`，也就是只能存入字符串，所以自定义模板很有必要。添加配置类 `RedisCacheConfig.java`

```java
@Configuration
@AutoConfigureAfter(RedisAutoConfiguration.class)
public class RedisCacheConfig {

    @Bean
    public RedisTemplate<String, Serializable> redisCacheTemplate(LettuceConnectionFactory connectionFactory) {

        RedisTemplate<String, Serializable> template = new RedisTemplate<>();
        template.setKeySerializer(new StringRedisSerializer());
        template.setValueSerializer(new GenericJackson2JsonRedisSerializer());
        template.setConnectionFactory(connectionFactory);
        return template;
    }
}
```

测试类：

```java
@RestController
@RequestMapping("/user")
public class UserController {

    public static Logger logger = LogManager.getLogger(UserController.class);

    @Autowired
    private StringRedisTemplate stringRedisTemplate;

    @Autowired
    private RedisTemplate<String, Serializable> redisCacheTemplate;

    @RequestMapping("/test")
    public void test() {
        redisCacheTemplate.opsForValue().set("userkey", new User(1, "张三", 25));
        User user = (User) redisCacheTemplate.opsForValue().get("userkey");
        logger.info("当前获取对象：{}", user.toString());
    }
```

然后在浏览器访问，观察后台日志 http://localhost:8082/user/test 。

![图片](https://mmbiz.qpic.cn/mmbiz_png/iaIdQfEric9TzlTcnXg26t1Dia266foajMicKPgg6GZwgftdOp8IcbvicF7CYY3Mib3pGFt05msY0bC72aojheBnDfCw/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

### 使用 Spring Cache 集成 Redis

Spring Cache 具备很好的灵活性，不仅能够使用 SPEL（spring expression language）来定义缓存的 Key 和各种  Condition，还提供了开箱即用的缓存临时存储方案，也支持和主流的专业缓存如 EhCache、Redis、Guava 的集成。

定义接口 `UserService.java`

```java
public interface UserService {

    User save(User user);

    void delete(int id);

    User get(Integer id);
}
```

接口实现类 `UserServiceImpl.java`

```java
@Service
public class UserServiceImpl implements UserService{

    public static Logger logger = LogManager.getLogger(UserServiceImpl.class);

    private static Map<Integer, User> userMap = new HashMap<>();
    static {
        userMap.put(1, new User(1, "肖战", 25));
        userMap.put(2, new User(2, "王一博", 26));
        userMap.put(3, new User(3, "杨紫", 24));
    }


    @CachePut(value ="user", key = "#user.id")
    @Override
    public User save(User user) {
        userMap.put(user.getId(), user);
        logger.info("进入save方法，当前存储对象：{}", user.toString());
        return user;
    }

    @CacheEvict(value="user", key = "#id")
    @Override
    public void delete(int id) {
        userMap.remove(id);
        logger.info("进入delete方法，删除成功");
    }

    @Cacheable(value = "user", key = "#id")
    @Override
    public User get(Integer id) {
        logger.info("进入get方法，当前获取对象：{}", userMap.get(id)==null?null:userMap.get(id).toString());
        return userMap.get(id);
    }
}
```

为了方便演示数据库的操作，这里直接定义了一个`Map<Integer,User> userMap`。

这里的核心是三个注解：

- **`@Cachable`**
- **`@CachePut`**
- **`@CacheEvict`**

测试类：`UserController`

```java
@RestController
@RequestMapping("/user")
public class UserController {

    public static Logger logger = LogManager.getLogger(UserController.class);

    @Autowired
    private StringRedisTemplate stringRedisTemplate;

    @Autowired
    private RedisTemplate<String, Serializable> redisCacheTemplate;

    @Autowired
    private UserService userService;

    @RequestMapping("/test")
    public void test() {
        redisCacheTemplate.opsForValue().set("userkey", new User(1, "张三", 25));
        User user = (User) redisCacheTemplate.opsForValue().get("userkey");
        logger.info("当前获取对象：{}", user.toString());
    }


    @RequestMapping("/add")
    public void add() {
        User user = userService.save(new User(4, "李现", 30));
        logger.info("添加的用户信息：{}",user.toString());
    }

    @RequestMapping("/delete")
    public void delete() {
        userService.delete(4);
    }

    @RequestMapping("/get/{id}")
    public void get(@PathVariable("id") String idStr) throws Exception{
        if (StringUtils.isBlank(idStr)) {
            throw new Exception("id为空");
        }
        Integer id = Integer.parseInt(idStr);
        User user = userService.get(id);
        logger.info("获取的用户信息：{}",user.toString());
    }
}
```

用缓存要注意，启动类要加上一个注解开启缓存：

```java
@SpringBootApplication(exclude=DataSourceAutoConfiguration.class)
@EnableCaching
public class Application {

    public static void main(String[] args) {
        SpringApplication.run(Application.class, args);
    }

}
```

1、先调用添加接口：http://localhost:8082/user/add

![图片](https://mmbiz.qpic.cn/mmbiz_png/iaIdQfEric9TzlTcnXg26t1Dia266foajMickTjd9sMMcqa12oZcEwyX8DON4ZY06jzdDQsFS1xpjIyO9emCIJIksA/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

2、再调用查询接口，查询 id=4 的用户信息：

![图片](https://mmbiz.qpic.cn/mmbiz_png/iaIdQfEric9TzlTcnXg26t1Dia266foajMicCoQMnNiaxmBxyBRcjKHSkU1a2Bdvlwg8fD3QfMyafxSFCvTHNIpycSw/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

可以看出，这里已经从缓存中获取数据了，因为上一步 add 方法已经把 id=4 的用户数据放入了 redis 缓存 3、调用删除方法，删除 id=4 的用户信息，同时清除缓存

![图片](https://mmbiz.qpic.cn/mmbiz_png/iaIdQfEric9TzlTcnXg26t1Dia266foajMicQA12AiaboqAvdBUwsyZdpNick3kXCRQ2TwNpHYxibxxgpWsiaKDoobjSgg/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

4、再次调用查询接口，查询 id=4 的用户信息：

![图片](https://mmbiz.qpic.cn/mmbiz_png/iaIdQfEric9TzlTcnXg26t1Dia266foajMicVCibQCZOjpNsejaCQF8mDB6rlVvSKkBmyRl6NtCEBR4Hs0tialM4uKsQ/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

没有了缓存，所以进入了 get 方法，从 userMap 中获取。

#### 缓存注解

1、`@Cacheable` 根据方法的请求参数对其结果进行缓存

- key：缓存的 key，可以为空，如果指定要按照 SPEL 表达式编写，如果不指定，则按照方法的所有参数进行组合。
- value：缓存的名称，必须指定至少一个（如 @Cacheable (value='user')或者@Cacheable(value={'user1','user2'})）
- condition：缓存的条件，可以为空，使用 SPEL 编写，返回 true 或者 false，只有为 true 才进行缓存。

> `@Cacheable` 注解不支持配置过期时间，所有需要通过配置 **cacheManager**来配置默认的过期时间和针对每个类或者是方法进行缓存失效时间配置。

2、`@CachePut`根据方法的请求参数对其结果进行缓存，和@Cacheable 不同的是，它每次都会触发真实方法的调用。参数描述见上。

3、`@CacheEvict`根据条件对缓存进行清空

- key：同上
- value：同上
- condition：同上
- allEntries：是否清空所有缓存内容，缺省为 false，如果指定为 true，则方法调用后将立即清空所有缓存
- beforeInvocation：是否在方法执行前就清空，缺省为 false，如果指定为 true，则在方法还没有执行的时候就清空缓存。缺省情况下，如果方法执行抛出异常，则不会清空缓存。

### 缓存问题

🙍‍♂️**面试官**：看了一下你的 demo，简单易懂。那你在实际项目中使用缓存有遇到什么问题或者会遇到什么问题你知道吗？

**🙋 我：** ：**缓存和数据库数据一致性问题**：分布式环境下非常容易出现缓存和数据库间数据一致性问题，针对这一点，如果项目对缓存的要求是强一致性的，那么就不要使用缓存。我们只能采取合适的策略来降低缓存和数据库间数据不一致的概率，而无法保证两者间的强一致性。合适的策略包括合适的缓存更新策略，更新数据库后及时更新缓存、缓存失败时增加重试机制。

🙍‍♂️**面试官** : **Redis 雪崩**了解吗？

**🙋 我：**  ：我了解的，目前电商首页以及热点数据都会去做缓存，一般缓存都是定时任务去刷新，或者查不到之后去更新缓存的，定时任务刷新就有一个问题。举个栗子：如果首页所有 Key 的失效时间都是 12 小时，中午 12 点刷新的，我零点有个大促活动大量用户涌入，假设每秒 6000 个请求，本来缓存可以抗住每秒  5000 个请求，但是缓存中所有 Key 都失效了。此时 6000 个/秒的请求全部落在了数据库上，数据库必然扛不住，真实情况可能 DBA  都没反应过来直接挂了，此时，如果没什么特别的方案来处理，DBA 很着急，重启数据库，但是数据库立马又被新流量给打死了。这就是我理解的缓存雪崩。

我心想：同一时间大面积失效，瞬间 Redis  跟没有一样，那这个数量级别的请求直接打到数据库几乎是灾难性的，你想想如果挂的是一个用户服务的库，那其他依赖他的库所有接口几乎都会报错，如果没做熔断等策略基本上就是瞬间挂一片的节奏，你怎么重启用户都会把你打挂，等你重启好的时候，用户早睡觉去了，临睡之前，骂骂咧咧“什么垃圾产品”。

🙍‍♂️**面试官** ：嗯，还不错，那这种情况你都是怎么应对的？（面试官摸摸了自己的头发）

**🙋 我**：处理缓存雪崩简单，在批量往 Redis 存数据的时候，把每个 Key 的失效时间都加个随机值就好了，这样可以保证数据不会再同一时间大面积失效。

```
setRedis（key, value, time+Math.random()*10000）;
```

如果 Redis 是集群部署，将热点数据均匀分布在不同的 Redis 库中也能避免全部失效。或者设置热点数据永不过期，有更新操作就更新缓存就好了（比如运维更新了首页商品，那你刷下缓存就好了，不要设置过期时间），电商首页的数据也可以用这个操作，保险。

**🙍‍♂️ 面试官**：那你了解**缓存穿透和击穿**么，可以说说他们跟雪崩的区别吗？

**🙋 我**：嗯，了解，先说下缓存穿透吧，缓存穿透是指缓存和数据库中都没有的数据，而用户（黑客）不断发起请求，举个栗子：我们数据库的 id 都是从 1 自增的，如果发起 id=-1 的数据或者 id 特别大不存在的数据，这样的不断攻击导致数据库压力很大，严重会击垮数据库。

**我又接着说**：至于缓存击穿嘛，这个跟缓存雪崩有点像，但是又有一点不一样，缓存雪崩是因为大面积的缓存失效，打崩了 DB，而缓存击穿不同的是缓存击穿是指一个 Key 非常热点，在不停地扛着大量的请求，大并发集中对这一个点进行访问，当这个 Key  在失效的瞬间，持续的大并发直接落到了数据库上，就在这个 Key 的点上击穿了缓存。

面试官露出欣慰的眼光：那他们分别怎么解决？

**🙋 我**：缓存穿透我会在接口层增加校验，比如用户鉴权，参数做校验，不合法的校验直接 return，比如 id 做基础校验，id<=0 直接拦截。

**🙍‍♂️ 面试官**：那你还有别的方法吗？

**🙋 我**：我记得 Redis 里还有一个高级用法**布隆过滤器（Bloom  Filter）**这个也能很好的预防缓存穿透的发生，他的原理也很简单，就是利用高效的数据结构和算法快速判断出你这个 Key  是否在数据库中存在，不存在你 return 就好了，存在你就去查 DB 刷新 KV 再  return。缓存击穿的话，设置热点数据永不过期，或者加上互斥锁就搞定了。作为暖男，代码给你准备好了，拿走不谢。

```java
public static String getData(String key) throws InterruptedException {
        //从Redis查询数据
        String result = getDataByKV(key);
        //参数校验
        if (StringUtils.isBlank(result)) {
            try {
                //获得锁
                if (reenLock.tryLock()) {
                    //去数据库查询
                    result = getDataByDB(key);
                    //校验
                    if (StringUtils.isNotBlank(result)) {
                        //插进缓存
                        setDataToKV(key, result);
                    }
                } else {
                    //睡一会再拿
                    Thread.sleep(100L);
                    result = getData(key);
                }
            } finally {
                //释放锁
                reenLock.unlock();
            }
        }
        return result;
    }
```

**面试官** ：嗯嗯，还不错。

## Redis 为何这么快

**🙍‍♂️ 面试官** ：redis 作为缓存大家都在用，那 redis 一定很快咯？

**🙋 我** ：当然了，官方提供的数据可以达到 100000+的 QPS（每秒内的查询次数），这个数据不比 Memcached 差！

**🙍‍♂️ 面试官** ：redis 这么快，它的“多线程模型”你了解吗？（露出邪魅一笑）

**🙋 我** ：您是想问 Redis 这么快，为什么还是单线程的吧。Redis 确实是单进程单线程的模型，因为 Redis 完全是基于内存的操作，CPU  不是 Redis 的瓶颈，Redis 的瓶颈最有可能是机器内存的大小或者网络带宽。既然单线程容易实现，而且 CPU  不会成为瓶颈，那就顺理成章的采用单线程的方案了（毕竟采用多线程会有很多麻烦）。

**🙍‍♂️ 面试官** ：嗯，是的。那你能说说 Redis 是单线程的，为什么还能这么快吗？

**🙋 我** ：可以这么说吧。第一：Redis 完全基于内存，绝大部分请求是纯粹的内存操作，非常迅速，数据存在内存中，类似于 HashMap，HashMap 的优势就是查找和操作的时间复杂度是  O(1)。第二：数据结构简单，对数据操作也简单。第三：采用单线程，避免了不必要的上下文切换和竞争条件，不存在多线程导致的 CPU  切换，不用去考虑各种锁的问题，不存在加锁释放锁操作，没有死锁问题导致的性能消耗。第四：使用多路复用 IO 模型，非阻塞 IO。

## Redis 和 Memcached 的区别

**🙍‍♂️ 面试官**：嗯嗯，说的很详细。那你为什么选择 Redis 的缓存方案而不用 memcached 呢

**🙋 我** ：

1. **存储方式上** ：memcache 会把数据全部存在内存之中，断电后会挂掉，数据不能超过内存大小。redis 有部分数据存在硬盘上，这样能保证数据的持久性。
2. **数据支持类型上** ：memcache 对数据类型的支持简单，只支持简单的 key-value，，而 redis 支持五种数据类型。
3. **使用底层模型不同：** 它们之间底层实现方式以及与客户端之间通信的应用协议不一样。redis 直接自己构建了 VM 机制，因为一般的系统调用系统函数的话，会浪费一定的时间去移动和请求。
4. **value 的大小** ：redis 可以达到 1GB，而 memcache 只有 1MB。

## 淘汰策略

**🙍‍♂️ 面试官** ：那你说说你知道的 redis 的淘汰策略有哪些？

**🙋 我** ：Redis 有六种淘汰策略

| 策略            | 描述                                                         |
| :-------------- | :----------------------------------------------------------- |
| volatile-lru    | 从已设置过期时间的 KV 集中优先对最近最少使用(less recently used)的数据淘汰 |
| volitile-ttl    | 从已设置过期时间的 KV 集中优先对剩余时间短(time to live)的数据淘汰 |
| volitile-random | 从已设置过期时间的 KV 集中随机选择数据淘汰                   |
| allkeys-lru     | 从所有 KV 集中优先对最近最少使用(less recently used)的数据淘汰 |
| allKeys-random  | 从所有 KV 集中随机选择数据淘汰                               |
| noeviction      | 不淘汰策略，若超过最大内存，返回错误信息                     |

4.0 版本后增加以下两种：

1. **volatile-lfu**：从已设置过期时间的数据集(server.db[i].expires)中挑选最不经常使用的数据淘汰
2. **allkeys-lfu**：当内存不足以容纳新写入数据时，在键空间中，移除最不经常使用的 key

## redis 持久化机制

**🙍‍♂️ 面试官** ：你对 redis 的持久化机制了解吗？怎么保证 redis 挂掉之后再重启数据可以进行恢复?能讲一下吗？

**🙋 我** ：redis  为了保证效率，数据缓存在了内存中，但是会周期性的把更新的数据写入磁盘或者把修改操作写入追加的记录文件中，以保证数据的持久化。Redis  的持久化策略有两种：1、RDB：快照形式是直接把内存中的数据保存到一个 dump 的文件中，定时保存，保存策略。2、AOF：把所有的对  Redis 的服务器进行修改的命令都存到一个文件里，命令的集合。Redis 默认是快照 RDB 的持久化方式。当 Redis  重启的时候，它会优先使用 AOF 文件来还原数据集，因为 AOF 文件保存的数据集通常比 RDB  文件所保存的数据集更完整。你甚至可以关闭持久化功能，让数据只在服务器运行时存。

**🙍‍♂️ 面试官** ：那你再说下 RDB 是怎么工作的？

**🙋 我** ：默认 Redis 是会以快照"RDB"的形式将数据持久化到磁盘的一个二进制文件 dump.rdb。工作原理简单说一下：当 Redis  需要做持久化时，Redis 会 fork 一个子进程，子进程将数据写到磁盘上一个临时 RDB 文件中。当子进程完成写临时文件后，将原来的 RDB 替换掉，这样的好处是可以 copy-on-write。

**🙋 我** ：RDB 的优点是：这种文件非常适合用于备份：比如，你可以在最近的 24 小时内，每小时备份一次，并且在每个月的每一天也备份一个 RDB  文件。这样的话，即使遇上问题，也可以随时将数据集还原到不同的版本。RDB 非常适合灾难恢复。RDB  的缺点是：如果你需要尽量避免在服务器故障时丢失数据，那么 RDB 不合适你。

**🙍‍♂️ 面试官** ：那你要不再说下 AOF？？

**🙋 我** ：（说就一起说下吧）使用 AOF 做持久化，每一个写命令都通过 write 函数追加到 appendonly.aof 中，配置方式如下：

```
appendfsync yes
appendfsync always     #每次有数据修改发生时都会写入AOF文件。
appendfsync everysec   #每秒钟同步一次，该策略为AOF的缺省策略。
```

AOF 可以做到全程持久化，只需要在配置中开启 appendonly yes。这样 redis 每执行一个修改数据的命令，都会把它添加到 AOF 文件中，当 redis 重启时，将会读取 AOF 文件进行重放，恢复到 redis 关闭前的最后时刻。

**🙋 我顿了一下，继续说** ：使用 AOF 的优点是会让 redis 变得非常耐久。可以设置不同的 fsync 策略，aof 的默认策略是每秒钟 fsync  一次，在这种配置下，就算发生故障停机，也最多丢失一秒钟的数据。缺点是对于相同的数据集来说，AOF 的文件体积通常要大于 RDB  文件的体积。根据所使用的 fsync 策略，AOF 的速度可能会慢于 RDB。

**🙍‍♂️ 面试官又问** ：你说了这么多，那我该用哪一个呢？

**🙋 我** ：如果你非常关心你的数据，但仍然可以承受数分钟内的数据丢失，那么可以额只使用 RDB 持久。AOF 将 Redis  执行的每一条命令追加到磁盘中，处理巨大的写入会降低 Redis 的性能，不知道你是否可以接受。数据库备份和灾难恢复：定时生成 RDB  快照非常便于进行数据库备份，并且 RDB 恢复数据集的速度也要比 AOF 恢复的速度快。当然了，redis 支持同时开启 RDB 和  AOF，系统重启后，redis 会优先使用 AOF 来恢复数据，这样丢失的数据会最少。

**Redis 4.0 对于持久化机制的优化**

Redis 4.0 开始支持 RDB 和 AOF 的混合持久化（默认关闭，可以通过配置项 `aof-use-rdb-preamble` 开启）。

如果把混合持久化打开，AOF 重写的时候就直接把 RDB 的内容写到 AOF 文件开头。这样做的好处是可以结合 RDB 和 AOF 的优点,  快速加载同时避免丢失过多的数据。当然缺点也是有的， AOF 里面的 RDB 部分是压缩格式不再是 AOF 格式，可读性较差。

**补充内容：AOF 重写**

AOF 重写可以产生一个新的 AOF 文件，这个新的 AOF 文件和原有的 AOF 文件所保存的数据库状态一样，但体积更小。

AOF 重写是一个有歧义的名字，该功能是通过读取数据库中的键值对来实现的，程序无须对现有 AOF 文件进行任何读入、分析或者写入操作。

在执行 BGREWRITEAOF 命令时，Redis 服务器会维护一个 AOF 重写缓冲区，该缓冲区会在子进程创建新 AOF  文件期间，记录服务器执行的所有写命令。当子进程完成创建新 AOF 文件的工作之后，服务器会将重写缓冲区中的所有内容追加到新 AOF  文件的末尾，使得新旧两个 AOF 文件所保存的数据库状态一致。最后，服务器用新的 AOF 文件替换旧的 AOF 文件，以此来完成 AOF  文件重写操作

## 主从复制

**🙍‍♂️ 面试官** ：redis 单节点存在单点故障问题，为了解决单点问题，一般都需要对 redis 配置从节点，然后使用哨兵来监听主节点的存活状态，如果主节点挂掉，从节点能继续提供缓存功能，你能说说 redis 主从复制的过程和原理吗？

**🙋 我** ：我有点懵，这个说来就话长了。但幸好提前准备了：主从配置结合哨兵模式能解决单点故障问题，提高 redis 可用性。从节点仅提供读操作，主节点提供写操作。对于读多写少的状况，可给主节点配置多个从节点，从而提高响应效率。

**🙋 我顿了一下，接着说：** 关于复制过程，是这样的：

1. 从节点执行`slaveof[masterIP][masterPort]`，保存主节点信息
2. 从节点中的定时任务发现主节点信息，建立和主节点的 socket 连接
3. 从节点发送 Ping 信号，主节点返回 Pong，两边能互相通信
4. 连接建立后，主节点将所有数据发送给从节点（数据同步）
5. 主节点把当前的数据同步给从节点后，便完成了复制的建立过程。接下来，主节点就会持续的把写命令发送给从节点，保证主从数据一致性。

**🙍‍♂️ 面试官** ：那你能详细说下数据同步的过程吗？

**🙋 我** ：（我心想：这也问的太细了吧）可以。redis2.8 之前使用`sync[runId][offset]`同步命令，redis2.8 之后使用`psync[runId][offset]`命令。两者不同在于，sync 命令仅支持全量复制过程，psync 支持全量和部分复制。介绍同步之前，先介绍几个概念：

- runId：每个 redis 节点启动都会生成唯一的 uuid，每次 redis 重启后，runId 都会发生变化。

- offset：主节点和从节点都各自维护自己的主从复制偏移量 offset，当主节点有写入命令时，offset=offset+命令的字节长度。从节点在收到主节点发送的命令后，也会增加自己的  offset，并把自己的 offset 发送给主节点。这样，主节点同时保存自己的 offset 和从节点的 offset，通过对比 offset 来判断主从节点数据是否一致。

- repl_backlog_size：保存在主节点上的一个固定长度的先进先出队列，默认大小是 1MB。

  （1）主节点发送数据给从节点过程中，主节点还会进行一些写操作，这时候的数据存储在复制缓冲区中。从节点同步主节点数据完成后，主节点将缓冲区的数据继续发送给从节点，用于部分复制。（2）主节点响应写命令时，不但会把命名发送给从节点，还会写入复制积压缓冲区，用于复制命令丢失的数据补救。

![图片](https://mmbiz.qpic.cn/mmbiz_png/iaIdQfEric9TzlTcnXg26t1Dia266foajMic6gIBl67EZnVQ8WSJRaCZRsHzvVYsticaVFtl2O4baBYAFcvrorNV13Q/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

上面是 psync 的执行流程：

从节点发送 psync[runId][offset]命令，主节点有三种响应：

1. **FULLRESYNC** ：第一次连接，进行全量复制
2. **CONTINUE** ：进行部分复制
3. **ERR** ：不支持 psync 命令，进行全量复制

**🙍‍♂️ 面试官** ：很好，那你能具体说下全量复制和部分复制的过程吗？

**🙋 我** ：可以

![图片](https://mmbiz.qpic.cn/mmbiz_png/iaIdQfEric9TzlTcnXg26t1Dia266foajMicVYLfZftxM7kQPHha4KRJlWUVtAsebwW7LBWyg7PyTyVDx0FdcEsY1Q/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

上面是全量复制的流程。主要有以下几步：

1. 从节点发送 psync ? -1 命令（因为第一次发送，不知道主节点的 runId，所以为?，因为是第一次复制，所以 offset=-1）。
2. 主节点发现从节点是第一次复制，返回 FULLRESYNC {runId} {offset}，runId 是主节点的 runId，offset 是主节点目前的 offset。
3. 从节点接收主节点信息后，保存到 info 中。
4. 主节点在发送 FULLRESYNC 后，启动 bgsave 命令，生成 RDB 文件（数据持久化）。
5. 主节点发送 RDB 文件给从节点。到从节点加载数据完成这段期间主节点的写命令放入缓冲区。
6. 从节点清理自己的数据库数据。
7. 从节点加载 RDB 文件，将数据保存到自己的数据库中。
8. 如果从节点开启了 AOF，从节点会异步重写 AOF 文件。

关于部分复制有以下几点说明：

1. 部分复制主要是 Redis 针对全量复制的过高开销做出的一种优化措施，使用`psync[runId][offset]`命令实现。当从节点正在复制主节点时，如果出现网络闪断或者命令丢失等异常情况时，从节点会向主节点要求补发丢失的命令数据，主节点的复制积压缓冲区将这部分数据直接发送给从节点，这样就可以保持主从节点复制的一致性。补发的这部分数据一般远远小于全量数据。
2. 主从连接中断期间主节点依然响应命令，但因复制连接中断命令无法发送给从节点，不过主节点内的复制积压缓冲区依然可以保存最近一段时间的写命令数据。
3. 当主从连接恢复后，由于从节点之前保存了自身已复制的偏移量和主节点的运行 ID。因此会把它们当做 psync 参数发送给主节点，要求进行部分复制。
4. 主节点接收到 psync 命令后首先核对参数 runId 是否与自身一致，如果一致，说明之前复制的是当前主节点；之后根据参数 offset  在复制积压缓冲区中查找，如果 offset 之后的数据存在，则对从节点发送+COUTINUE  命令，表示可以进行部分复制。因为缓冲区大小固定，若发生缓冲溢出，则进行全量复制。
5. 主节点根据偏移量把复制积压缓冲区里的数据发送给从节点，保证主从复制进入正常状态。

## 哨兵

**🙍‍♂️ 面试官** ：那主从复制会存在哪些问题呢？

**🙋 我** ：主从复制会存在以下问题：

1. 一旦主节点宕机，从节点晋升为主节点，同时需要修改应用方的主节点地址，还需要命令所有从节点去复制新的主节点，整个过程需要人工干预。
2. 主节点的写能力受到单机的限制。
3. 主节点的存储能力受到单机的限制。
4. 原生复制的弊端在早期的版本中也会比较突出，比如：redis 复制中断后，从节点会发起 psync。此时如果同步不成功，则会进行全量同步，主库执行全量备份的同时，可能会造成毫秒或秒级的卡顿。

**🙍‍♂️ 面试官** ：那比较主流的解决方案是什么呢？

**🙋 我** ：当然是哨兵啊。

**🙍‍♂️ 面试官** ：那么问题又来了。那你说下哨兵有哪些功能？

![图片](https://mmbiz.qpic.cn/mmbiz_png/iaIdQfEric9TzlTcnXg26t1Dia266foajMic89F6770xHiaYPJN48zJR2LB8A6aP3VfIgC0vVxibVlYicy2gwiaqXdSrPw/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

**🙋 我** ：如图，是 Redis Sentinel（哨兵）的架构图。Redis  Sentinel（哨兵）主要功能包括主节点存活检测、主从运行情况检测、自动故障转移、主从切换。Redis Sentinel  最小配置是一主一从。Redis 的 Sentinel 系统可以用来管理多个 Redis 服务器，该系统可以执行以下四个任务：

1. **监控** ：不断检查主服务器和从服务器是否正常运行。
2. **通知** ：当被监控的某个 redis 服务器出现问题，Sentinel 通过 API 脚本向管理员或者其他应用程序发出通知。
3. **自动故障转移** ：当主节点不能正常工作时，Sentinel 会开始一次自动的故障转移操作，它会将与失效主节点是主从关系的其中一个从节点升级为新的主节点，并且将其他的从节点指向新的主节点，这样人工干预就可以免了。
4. **配置提供者** ：在 Redis Sentinel 模式下，客户端应用在初始化时连接的是 Sentinel 节点集合，从中获取主节点的信息。

**🙍‍♂️ 面试官** ：那你能说下哨兵的工作原理吗？

**🙋 我** ：话不多说，直接上图：

![图片](https://mmbiz.qpic.cn/mmbiz_png/iaIdQfEric9TzlTcnXg26t1Dia266foajMicV4uRLib3FmS9KibcSMycB36MwicA3GTygLnQTl3VkAGb8mPE47pLzcz0g/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

1、每个 Sentinel 节点都需要定期执行以下任务：每个 Sentinel 以每秒一次的频率，向它所知的主服务器、从服务器以及其他的 Sentinel 实例发送一个 PING 命令。（如上图）

![图片](https://mmbiz.qpic.cn/mmbiz_png/iaIdQfEric9TzlTcnXg26t1Dia266foajMicexruL5YRczic7adyYa4eHvmDhTCYkdBsgia7wNExsIp3iaENcCwhYztYg/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

2、如果一个实例距离最后一次有效回复 PING 命令的时间超过`down-after-milliseconds`所指定的值，那么这个实例会被 Sentinel 标记为主观下线。（如上图）

![图片](https://mmbiz.qpic.cn/mmbiz_png/iaIdQfEric9TzlTcnXg26t1Dia266foajMickrEiadYfFM6Fq55GTUCSibTzLMib5xn7200NFu2J56iadM1vSCiaHy04xXg/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

3、如果一个主服务器被标记为主观下线，那么正在监视这个服务器的所有 Sentinel 节点，要以每秒一次的频率确认主服务器的确进入了主观下线状态。

![图片](https://mmbiz.qpic.cn/mmbiz_png/iaIdQfEric9TzlTcnXg26t1Dia266foajMicgeP6wxMBPbUyutWY0JXyADiaHXDNtZOkDX3NZqagTdcjiabBJ3TYwW5w/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

4、如果一个主服务器被标记为主观下线，并且有足够数量的 Sentinel（至少要达到配置文件指定的数量）在指定的时间范围内同意这一判断，那么这个主服务器被标记为客观下线。

![图片](https://mmbiz.qpic.cn/mmbiz_png/iaIdQfEric9TzlTcnXg26t1Dia266foajMicyb8R8NQYSlO3HQ5Y1ZPpKVA0QkCt44eFIDSmAX1iczcsr8eKYPlfZgQ/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

5、一般情况下，每个 Sentinel 会以每 10 秒一次的频率向它已知的所有主服务器和从服务器发送 INFO  命令，当一个主服务器被标记为客观下线时，Sentinel 向下线主服务器的所有从服务器发送 INFO 命令的频率，会从 10  秒一次改为每秒一次。

![图片](https://mmbiz.qpic.cn/mmbiz_png/iaIdQfEric9TzlTcnXg26t1Dia266foajMicFYTOiaPwr2eTzrOwBaFcSMqia8CGvpt20RLL0hoCXY08PI1z1dDG3S5Q/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

6、Sentinel 和其他 Sentinel 协商客观下线的主节点的状态，如果处于 SDOWN 状态，则投票自动选出新的主节点，将剩余从节点指向新的主节点进行数据复制。

![图片](https://mmbiz.qpic.cn/mmbiz_png/iaIdQfEric9TzlTcnXg26t1Dia266foajMic6W1t4T0lTkkPmrm1ntavVFZFcCaTyMU4VqZLxGWias15icR40Bibh8GDA/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

7、当没有足够数量的 Sentinel 同意主服务器下线时，主服务器的客观下线状态就会被移除。当主服务器重新向 Sentinel 的 PING 命令返回有效回复时，主服务器的主观下线状态就会被移除。

**Redis常见的几种主要使用方式：**

- Redis 单副本
- Redis 多副本（主从）
- Redis Sentinel（哨兵）
- Redis Cluster
- Redis 自研



## Redis各种使用方式的优缺点：

1

Redis单副本

![图片](https://mmbiz.qpic.cn/mmbiz_png/6EJvicazJ6KzImNoKH1nofWM9rqIR6VUWtkGVXibFND01jgOkPqJ9iajeiby7jCO8tLFsZt4eNg9kcw8MNgqlzU19Q/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

Redis 单副本，采用单个Redis节点部署架构，没有备用节点实时同步数据，不提供数据持久化和备份策略，适用于数据可靠性要求不高的纯缓存业务场景。



**优点：**

1、架构简单、部署方便

 2、高性价比，当缓存使用时无需备用节点（单实例可用性可以用supervisor或crontab保证），当然为了满足业务的高可用性，也可以牺牲一个备用节点，但同时刻只有一个实例对外提供服务。

 3、高性能



**缺点：**

1、不保证数据的可靠性

 2、当缓存使用，进程重启后，数据丢失，即使有备用的节点解决高可用性，但是仍然不能解决缓存预热问题，因此不适用于数据可靠性要求高的业务。

3、高性能受限于单核CPU的处理能力（Redis是单线程机制），CPU为主要瓶颈，所以适合操作命令简单，排序、计算较少的场景。也可以考虑用memcached替代。



2

Redis多副本（主从）

![图片](https://mmbiz.qpic.cn/mmbiz_png/6EJvicazJ6KzImNoKH1nofWM9rqIR6VUWeDxJHhJHIQgDFF9Hb5kRnGkNicM7KGfpad5VnUo0pOQjm2Fw7TGUs0Q/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)Redis 多副本，采用主从（replication）部署结构，相较于单副本而言最大的特点就是主从实例间数据实时同步，并且提供数据持久化和备份策略。主从实例部署在不同的物理服务器上，根据公司的基础环境配置，可以实现同时对外提供服务和读写分离策略。

优点：

1、高可靠性，一方面，采用双机主备架构，能够在主库出现故障时自动进行主备切换，从库提升为主库提供服务，保证服务平稳运行。另一方面，开启数据持久化功能和配置合理的备份策略，能有效的解决数据误操作和数据异常丢失的问题。

2、读写分离策略，从节点可以扩展主库节点的读能力，有效应对大并发量的读操作。



缺点：

1、故障恢复复杂，如果没有RedisHA系统（需要开发），当主库节点出现故障时，需要手动将一个从节点晋升为主节点，同时需要通知业务方变更配置，并且需要让其他从库节点去复制新主库节点，整个过程需要人为干预，比较繁琐。

2、主库的写能力受到单机的限制，可以考虑分片

3、主库的存储能力受到单机的限制，可以考虑Pika

4、原生复制的弊端在早期的版本也会比较突出，如：Redis复制中断后，Slave会发起psync，此时如果同步不成功，则会进行全量同步，主库执行全量备份的同时可能会造成毫秒或秒级的卡顿；又由于COW机制，导致极端情况下的主库内存溢出，程序异常退出或宕机；主库节点生成备份文件导致服务器磁盘IO和CPU（压缩）资源消耗；发送数GB大小的备份文件导致服务器出口带宽暴增，阻塞请求。建议升级到最新版本。



3

Redis Sentinel（哨兵）

![图片](https://mmbiz.qpic.cn/mmbiz_png/6EJvicazJ6KzImNoKH1nofWM9rqIR6VUWfIC0vFhMAib4z10k7YqO8QJHTDxnujrrkvgjeibBmhV78vAAGzNfplCw/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

![图片](https://mmbiz.qpic.cn/mmbiz_png/6EJvicazJ6KzImNoKH1nofWM9rqIR6VUW1ibIW7Go066hNFQF5jUpkv5NN8RIA4McUSyRPVb82OTnpnWnmKyrDbA/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

Redis Sentinel是社区版本推出的原生高可用解决方案，Redis Sentinel部署架构主要包括两部分：Redis  Sentinel集群和Redis数据集群，其中Redis  Sentinel集群是由若干Sentinel节点组成的分布式集群。可以实现故障发现、故障自动转移、配置中心和客户端通知。Redis  Sentinel的节点数量要满足2n+1（n>=1）的奇数个。



优点：

1、Redis Sentinel集群部署简单

2、能够解决Redis主从模式下的高可用切换问题

3、很方便实现Redis数据节点的线形扩展，轻松突破Redis自身单线程瓶颈，可极大满足对Redis大容量或高性能的业务需求。

4、可以实现一套Sentinel监控一组Redis数据节点或多组数据节点



缺点：

1、部署相对Redis 主从模式要复杂一些，原理理解更繁琐

2、资源浪费，Redis数据节点中slave节点作为备份节点不提供服务

3、Redis Sentinel主要是针对Redis数据节点中的主节点的高可用切换，对Redis的数据节点做失败判定分为主观下线和客观下线两种，对于Redis的从节点有对节点做主观下线操作，并不执行故障转移。

4、不能解决读写分离问题，实现起来相对复杂



建议：

1、如果监控同一业务，可以选择一套Sentinel集群监控多组Redis数据节点的方案，反之选择一套Sentinel监控一组Redis数据节点的方案

2、sentinel monitor <master-name> <ip> <port> <quorum>  配置中的<quorum>建议设置成Sentinel节点的一半加1，当Sentinel部署在多个IDC的时候，单个IDC部署的Sentinel数量不建议超过（Sentinel数量 – quorum）。

3、合理设置参数，防止误切，控制切换灵敏度控制

1. quorum
2. down-after-milliseconds 30000
3. failover-timeout 180000
4. maxclient
5. timeout

4、部署的各个节点服务器时间尽量要同步，否则日志的时序性会混乱

5、Redis建议使用pipeline和multi-keys操作，减少RTT次数，提高请求效率

6、自行搞定配置中心（zookeeper），方便客户端对实例的链接访问



4

Redis Cluster

![图片](https://mmbiz.qpic.cn/mmbiz_png/6EJvicazJ6KzImNoKH1nofWM9rqIR6VUWNSEC8dvIoHtWlm4OyK3KxaXc4sSzibK0nFC4hNKhhsPXibrQibFwLOglA/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

Redis  Cluster是社区版推出的Redis分布式集群解决方案，主要解决Redis分布式方面的需求，比如，当遇到单机内存，并发和流量等瓶颈的时候，Redis Cluster能起到很好的负载均衡的目的。Redis  Cluster集群节点最小配置6个节点以上（3主3从），其中主节点提供读写操作，从节点作为备用节点，不提供请求，只作为故障转移使用。Redis  Cluster采用虚拟槽分区，所有的键根据哈希函数映射到0～16383个整数槽内，每个节点负责维护一部分槽以及槽所印映射的键值数据。



优点：

1、无中心架构

2、数据按照slot存储分布在多个节点，节点间数据共享，可动态调整数据分布。

3、可扩展性，可线性扩展到1000多个节点，节点可动态添加或删除。

4、高可用性，部分节点不可用时，集群仍可用。通过增加Slave做standby数据副本，能够实现故障自动failover，节点之间通过gossip协议交换状态信息，用投票机制完成Slave到Master的角色提升。

5、降低运维成本，提高系统的扩展性和可用性。



缺点：

1、Client实现复杂，驱动要求实现Smart Client，缓存slots  mapping信息并及时更新，提高了开发难度，客户端的不成熟影响业务的稳定性。目前仅JedisCluster相对成熟，异常处理部分还不完善，比如常见的“max redirect exception”。

2、节点会因为某些原因发生阻塞（阻塞时间大于clutser-node-timeout），被判断下线，这种failover是没有必要的。

3、数据通过异步复制,不保证数据的强一致性。

4、多个业务使用同一套集群时，无法根据统计区分冷热数据，资源隔离性较差，容易出现相互影响的情况。

5、Slave在集群中充当“冷备”，不能缓解读压力，当然可以通过SDK的合理设计来提高Slave资源的利用率。

6、key批量操作限制，如使用mset、mget目前只支持具有相同slot值的key执行批量操作。对于映射为不同slot值的key由于keys 不支持跨slot查询，所以执行mset、mget、sunion等操作支持不友好。

7、key事务操作支持有限，只支持多key在同一节点上的事务操作，当多个key分布于不同的节点上时无法使用事务功能。

8、key作为数据分区的最小粒度，因此不能将一个很大的键值对象如hash、list等映射到不同的节点。

9、不支持多数据库空间，单机下的redis可以支持到16个数据库，集群模式下只能使用1个数据库空间，即db 0。

10、复制结构只支持一层，从节点只能复制主节点，不支持嵌套树状复制结构。

11、避免产生hot-key，导致主库节点成为系统的短板。

12、避免产生big-key，导致网卡撑爆、慢查询等。

13、重试时间应该大于cluster-node-time时间

14、Redis Cluster不建议使用pipeline和multi-keys操作，减少max redirect产生的场景。



5

Redis自研 - 推荐

![图片](https://mmbiz.qpic.cn/mmbiz_png/6EJvicazJ6KzImNoKH1nofWM9rqIR6VUWF5ztvp6RiabrQWvWBPH2pVgibgaAPQ5FLddA0htYRy4kxgz7cgicvY4Rw/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

![图片](https://mmbiz.qpic.cn/mmbiz_png/6EJvicazJ6KzImNoKH1nofWM9rqIR6VUW2MdMwUDDibguJCWQrGFeJTE9znLEe1h7wsLFb94MRsjnmWwcdzWeHuA/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

Redis 自研的高可用解决方案，主要体现在配置中心、故障探测和failover的处理机制上，通常需要根据企业业务的实际线上环境来定制化。



优点：

1、高可靠性、高可用性

2、自主可控性高

3、贴切业务实际需求，可缩性好，兼容性好



缺点：

1、实现复杂，开发成本高

2、需要建立配套的周边设施，如监控，域名服务，存储元数据信息的数据库等。

3、维护成本高



# 秒杀系统设计            

![img](https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2019/11/16/16e72d24415cb009~tplv-t2oaga2asx-watermark.image)

跳表？