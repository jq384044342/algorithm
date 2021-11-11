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

#### 基于单Redis节点的分布式锁





首先，Redis客户端为了**获取锁**，向Redis节点发送如下命令：

```
SET resource_name my_random_value NX PX 30000
```

上面的命令如果执行成功，则客户端成功获取到了锁，接下来就可以**访问共享资源**了；而如果上面的命令执行失败，则说明获取锁失败。

注意，在上面的`SET`命令中：

- `my_random_value`是由客户端生成的一个随机字符串，它要保证在足够长的一段时间内在所有客户端的所有获取锁的请求中都是唯一的。
- `NX`表示只有当`resource_name`对应的key值不存在的时候才能`SET`成功。这保证了只有第一个请求的客户端才能获得锁，而其它客户端在锁被释放之前都无法获得锁。
- `PX 30000`表示这个锁有一个30秒的自动过期时间。当然，这里30秒只是一个例子，客户端可以选择合适的过期时间。

最后，当客户端完成了对共享资源的操作之后，执行下面的Redis Lua脚本来**释放锁**：

```
if redis.call("get",KEYS[1]) == ARGV[1] then
    return redis.call("del",KEYS[1])
else
    return 0
end
```

这段Lua脚本在执行的时候要把前面的`my_random_value`作为`ARGV[1]`的值传进去，把`resource_name`作为`KEYS[1]`的值传进去。

至此，基于单Redis节点的分布式锁的算法就描述完了。这里面有好几个问题需要重点分析一下。

首先第一个问题，这个锁必须要设置一个过期时间。否则的话，当一个客户端获取锁成功之后，假如它崩溃了，或者由于发生了网络分割（network  partition）导致它再也无法和Redis节点通信了，那么它就会一直持有这个锁，而其它客户端永远无法获得锁了。antirez在后面的分析中也特别强调了这一点，而且把这个过期时间称为锁的有效时间(lock validity time)。获得锁的客户端必须在这个时间之内完成对共享资源的访问。

第二个问题，第一步**获取锁**的操作，网上不少文章把它实现成了两个Redis命令：

```
SETNX resource_name my_random_value
EXPIRE resource_name 30
```

虽然这两个命令和前面算法描述中的一个`SET`命令执行效果相同，但却不是原子的。如果客户端在执行完`SETNX`后崩溃了，那么就没有机会执行`EXPIRE`了，导致它一直持有这个锁。

第三个问题，也是antirez指出的，设置一个随机字符串`my_random_value`是很有必要的，它保证了一个客户端释放的锁必须是自己持有的那个锁。假如获取锁时`SET`的不是一个随机字符串，而是一个固定值，那么可能会发生下面的执行序列：

1. 客户端1获取锁成功。
2. 客户端1在某个操作上阻塞了很长时间。
3. 过期时间到了，锁自动释放了。
4. 客户端2获取到了对应同一个资源的锁。
5. 客户端1从阻塞中恢复过来，释放掉了客户端2持有的锁。

之后，客户端2在访问共享资源的时候，就没有锁为它提供保护了。

第四个问题，释放锁的操作必须使用Lua脚本来实现。释放锁其实包含三步操作：'GET'、判断和'DEL'，用Lua脚本来实现能保证这三步的原子性。否则，如果把这三步操作放到客户端逻辑中去执行的话，就有可能发生与前面第三个问题类似的执行序列：

1. 客户端1获取锁成功。
2. 客户端1访问共享资源。
3. 客户端1为了释放锁，先执行'GET'操作获取随机字符串的值。
4. 客户端1判断随机字符串的值，与预期的值相等。
5. 客户端1由于某个原因阻塞住了很长时间。
6. 过期时间到了，锁自动释放了。
7. 客户端2获取到了对应同一个资源的锁。
8. 客户端1从阻塞中恢复过来，执行`DEL`操纵，释放掉了客户端2持有的锁。

实际上，在上述第三个问题和第四个问题的分析中，如果不是客户端阻塞住了，而是出现了大的网络延迟，也有可能导致类似的执行序列发生。

前面的四个问题，只要实现分布式锁的时候加以注意，就都能够被正确处理。但除此之外，antirez还指出了一个问题，是由failover引起的，却是基于单Redis节点的分布式锁无法解决的。正是这个问题催生了Redlock的出现。

这个问题是这样的。假如Redis节点宕机了，那么所有客户端就都无法获得锁了，服务变得不可用。为了提高可用性，我们可以给这个Redis节点挂一个Slave，当Master节点不可用的时候，系统自动切到Slave上（failover）。但由于Redis的主从复制（replication）是异步的，这可能导致在failover过程中丧失锁的安全性。考虑下面的执行序列：

1. 客户端1从Master获取了锁。
2. Master宕机了，存储锁的key还没有来得及同步到Slave上。
3. Slave升级为Master。
4. 客户端2从新的Master获取到了对应同一个资源的锁。

于是，客户端1和客户端2同时持有了同一个资源的锁。锁的安全性被打破。针对这个问题，antirez设计了Redlock算法，我们接下来会讨论。

#### 分布式锁Redlock





由于前面介绍的基于单Redis节点的分布式锁在failover的时候会产生解决不了的安全性问题，因此antirez提出了新的分布式锁的算法Redlock，它基于N个完全独立的Redis节点（通常情况下N可以设置成5）。

运行Redlock算法的客户端依次执行下面各个步骤，来完成**获取锁**的操作：

1. 获取当前时间（毫秒数）。
2. 按顺序依次向N个Redis节点执行**获取锁**的操作。这个获取操作跟前面基于单Redis节点的**获取锁**的过程相同，包含随机字符串`my_random_value`，也包含过期时间(比如`PX 30000`，即锁的有效时间)。为了保证在某个Redis节点不可用的时候算法能够继续运行，这个**获取锁**的操作还有一个超时时间(time  out)，它要远小于锁的有效时间（几十毫秒量级）。客户端在向某个Redis节点获取锁失败以后，应该立即尝试下一个Redis节点。这里的失败，应该包含任何类型的失败，比如该Redis节点不可用，或者该Redis节点上的锁已经被其它客户端持有（注：Redlock原文中这里只提到了Redis节点不可用的情况，但也应该包含其它的失败情况）。
3. 计算整个获取锁的过程总共消耗了多长时间，计算方法是用当前时间减去第1步记录的时间。如果客户端从大多数Redis节点（>= N/2+1）成功获取到了锁，并且获取锁总共消耗的时间没有超过锁的有效时间(lock validity  time)，那么这时客户端才认为最终获取锁成功；否则，认为最终获取锁失败。
4. 如果最终获取锁成功了，那么这个锁的有效时间应该重新计算，它等于最初的锁的有效时间减去第3步计算出来的获取锁消耗的时间。
5. 如果最终获取锁失败了（可能由于获取到锁的Redis节点个数少于N/2+1，或者整个获取锁的过程消耗的时间超过了锁的最初有效时间），那么客户端应该立即向所有Redis节点发起**释放锁**的操作（即前面介绍的Redis Lua脚本）。

当然，上面描述的只是**获取锁**的过程，而**释放锁**的过程比较简单：客户端向所有Redis节点发起**释放锁**的操作，不管这些节点当时在获取锁的时候成功与否。

**watchdog**

也叫做看门狗，也就是解决了锁超时导致的问题，实际上就是一个后台线程，默认每隔10秒自动延长锁的过期时间。

默认的时间就是`internalLockLeaseTime / 3`，`internalLockLeaseTime`默认为30秒。

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/ibBMVuDfkZUkr0NZvdnibzGXkPiaucEtNLZFJ9oEDU1MhjPp4NRG330YwQj242MfgxPImklxUVfcbWgU5dMMRuoKQ/640?wx_fmt=jpeg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

### 最后，实际生产中对于不同的场景该如何选择？

由于N个Redis节点中的大多数能正常工作就能保证Redlock正常工作，因此理论上它的可用性更高。我们前面讨论的单Redis节点的分布式锁在failover的时候锁失效的问题，在Redlock中不存在了，但如果有节点发生崩溃重启，还是会对锁的安全性有影响的。具体的影响程度跟Redis对数据的持久化程度有关。

假设一共有5个Redis节点：A, B, C, D, E。设想发生了如下的事件序列：

1. 客户端1成功锁住了A, B, C，**获取锁**成功（但D和E没有锁住）。
2. 节点C崩溃重启了，但客户端1在C上加的锁没有持久化下来，丢失了。
3. 节点C重启后，客户端2锁住了C, D, E，**获取锁**成功。

这样，客户端1和客户端2同时获得了锁（针对同一资源）。

在默认情况下，Redis的AOF持久化方式是每秒写一次磁盘（即执行fsync），因此最坏情况下可能丢失1秒的数据。为了尽可能不丢数据，Redis允许设置成每次修改数据都进行fsync，但这会降低性能。当然，即使执行了fsync也仍然有可能丢失数据（这取决于系统而不是Redis的实现）。所以，上面分析的由于节点重启引发的锁失效问题，总是有可能出现的。为了应对这一问题，antirez又提出了**延迟重启**(delayed restarts)的概念。也就是说，一个节点崩溃后，先不立即重启它，而是等待一段时间再重启，这段时间应该大于锁的有效时间(lock  validity time)。这样的话，这个节点在重启前所参与的锁都会过期，它在重启后就不会对现有的锁造成影响。

关于Redlock还有一点细节值得拿出来分析一下：在最后**释放锁**的时候，antirez在算法描述中特别强调，客户端应该向所有Redis节点发起**释放锁**的操作。也就是说，即使当时向某个节点获取锁没有成功，在释放锁的时候也不应该漏掉这个节点。这是为什么呢？设想这样一种情况，客户端发给某个Redis节点的**获取锁**的请求成功到达了该Redis节点，这个节点也成功执行了`SET`操作，但是它返回给客户端的响应包却丢失了。这在客户端看来，获取锁的请求由于超时而失败了，但在Redis这边看来，加锁已经成功了。因此，释放锁的时候，客户端也应该对当时获取锁失败的那些Redis节点同样发起请求。实际上，这种情况在异步通信模型中是有可能发生的：客户端向服务器通信是正常的，但反方向却是有问题的。

【***其它疑问\***】

前面在讨论单Redis节点的分布式锁的时候，最后我们提出了一个疑问，如果客户端长期阻塞导致锁过期，那么它接下来访问共享资源就不安全了（没有了锁的保护）。这个问题在Redlock中是否有所改善呢？显然，这样的问题在Redlock中是依然存在的。

另外，在算法第4步成功获取了锁之后，如果由于获取锁的过程消耗了较长时间，重新计算出来的剩余的锁有效时间很短了，那么我们还来得及去完成共享资源访问吗？如果我们认为太短，是不是应该立即进行锁的释放操作？那到底多短才算呢？又是一个选择难题。



### Martin的分析





Martin Kleppmann在2016-02-08这一天发表了一篇blog，名字叫"How to do distributed locking"，地址如下：

- https://martin.kleppmann.com/2016/02/08/how-to-do-distributed-locking.html

Martin在这篇文章中谈及了分布式系统的很多基础性的问题（特别是分布式计算的异步模型），对分布式系统的从业者来说非常值得一读。这篇文章大体可以分为两大部分：

- 前半部分，与Redlock无关。Martin指出，即使我们拥有一个完美实现的分布式锁（带自动过期功能），在没有共享资源参与进来提供某种fencing机制的前提下，我们仍然不可能获得足够的安全性。
- 后半部分，是对Redlock本身的批评。Martin指出，由于Redlock本质上是建立在一个同步模型之上，对系统的记时假设(timing assumption)有很强的要求，因此本身的安全性是不够的。

首先我们讨论一下前半部分的关键点。Martin给出了下面这样一份时序图：

![图片](http://mmbiz.qpic.cn/mmbiz_png/qR4rtg9Wfej8vSic8l7ICJyXuU2PwibtddBsbjPwOt2xod1YtoicVoVibbo8Jiaw2QdXmM2zBCbCcghrK8VGENIicZ6w/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

在上面的时序图中，假设锁服务本身是没有问题的，它总是能保证任一时刻最多只有一个客户端获得锁。上图中出现的lease这个词可以暂且认为就等同于一个带有自动过期功能的锁。客户端1在获得锁之后发生了很长时间的GC pause，在此期间，它获得的锁过期了，而客户端2获得了锁。当客户端1从GC  pause中恢复过来的时候，它不知道自己持有的锁已经过期了，它依然向共享资源（上图中是一个存储服务）发起了写数据请求，而这时锁实际上被客户端2持有，因此两个客户端的写请求就有可能冲突（锁的互斥作用失效了）。

初看上去，有人可能会说，既然客户端1从GC pause中恢复过来以后不知道自己持有的锁已经过期了，那么它可以在访问共享资源之前先判断一下锁是否过期。但仔细想想，这丝毫也没有帮助。因为GC pause可能发生在任意时刻，也许恰好在判断完之后。

也有人会说，如果客户端使用没有GC的语言来实现，是不是就没有这个问题呢？Martin指出，系统环境太复杂，仍然有很多原因导致进程的pause，比如虚存造成的缺页故障(page fault)，再比如CPU资源的竞争。即使不考虑进程pause的情况，网络延迟也仍然会造成类似的结果。

总结起来就是说，即使锁服务本身是没有问题的，而仅仅是客户端有长时间的pause或网络延迟，仍然会造成两个客户端同时访问共享资源的冲突情况发生。而这种情况其实就是我们在前面已经提出来的“客户端长期阻塞导致锁过期”的那个疑问。

那怎么解决这个问题呢？Martin给出了一种方法，称为fencing token。fencing  token是一个单调递增的数字，当客户端成功获取锁的时候它随同锁一起返回给客户端。而客户端访问共享资源的时候带着这个fencing  token，这样提供共享资源的服务就能根据它进行检查，拒绝掉延迟到来的访问请求（避免了冲突）。如下图：

![图片](http://mmbiz.qpic.cn/mmbiz_png/qR4rtg9Wfej8vSic8l7ICJyXuU2PwibtddYe5ALGjZntmclabVY714kNdRfTKtECXhibZVKlEIGseClQiaRLRUDRnA/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

在上图中，客户端1先获取到的锁，因此有一个较小的fencing token，等于33，而客户端2后获取到的锁，有一个较大的fencing token，等于34。客户端1从GC  pause中恢复过来之后，依然是向存储服务发送访问请求，但是带了fencing token =  33。存储服务发现它之前已经处理过34的请求，所以会拒绝掉这次33的请求。这样就避免了冲突。

现在我们再讨论一下Martin的文章的后半部分。

Martin在文中构造了一些事件序列，能够让Redlock失效（两个客户端同时持有锁）。为了说明Redlock对系统记时(timing)的过分依赖，他首先给出了下面的一个例子（还是假设有5个Redis节点A, B, C, D, E）：

1. 客户端1从Redis节点A, B, C成功获取了锁（多数节点）。由于网络问题，与D和E通信失败。
2. 节点C上的时钟发生了向前跳跃，导致它上面维护的锁快速过期。
3. 客户端2从Redis节点C, D, E成功获取了同一个资源的锁（多数节点）。
4. 客户端1和客户端2现在都认为自己持有了锁。

上面这种情况之所以有可能发生，本质上是因为Redlock的安全性(safety  property)对系统的时钟有比较强的依赖，一旦系统的时钟变得不准确，算法的安全性也就保证不了了。Martin在这里其实是要指出分布式算法研究中的一些基础性问题，或者说一些常识问题，即好的分布式算法应该基于异步模型(asynchronous model)，算法的安全性不应该依赖于任何记时假设(timing  assumption)。在异步模型中：进程可能pause任意长的时间，消息可能在网络中延迟任意长的时间，甚至丢失，系统时钟也可能以任意方式出错。一个好的分布式算法，这些因素不应该影响它的安全性(safety property)，只可能影响到它的活性(liveness  property)，也就是说，即使在非常极端的情况下（比如系统时钟严重错误），算法顶多是不能在有限的时间内给出结果而已，而不应该给出错误的结果。这样的算法在现实中是存在的，像比较著名的Paxos，或Raft。但显然按这个标准的话，Redlock的安全性级别是达不到的。

随后，Martin觉得前面这个时钟跳跃的例子还不够，又给出了一个由客户端GC pause引发Redlock失效的例子。如下：

1. 客户端1向Redis节点A, B, C, D, E发起锁请求。
2. 各个Redis节点已经把请求结果返回给了客户端1，但客户端1在收到请求结果之前进入了长时间的GC pause。
3. 在所有的Redis节点上，锁过期了。
4. 客户端2在A, B, C, D, E上获取到了锁。
5. 客户端1从GC pause从恢复，收到了前面第2步来自各个Redis节点的请求结果。客户端1认为自己成功获取到了锁。
6. 客户端1和客户端2现在都认为自己持有了锁。

Martin给出的这个例子其实有点小问题。在Redlock算法中，客户端在完成向各个Redis节点的获取锁的请求之后，会计算这个过程消耗的时间，然后检查是不是超过了锁的有效时间(lock validity time)。也就是上面的例子中第5步，客户端1从GC  pause中恢复过来以后，它会通过这个检查发现锁已经过期了，不会再认为自己成功获取到锁了。随后antirez在他的反驳文章中就指出来了这个问题，但Martin认为这个细节对Redlock整体的安全性没有本质的影响。

抛开这个细节，我们可以分析一下Martin举这个例子的意图在哪。初看起来，这个例子跟文章前半部分分析通用的分布式锁时给出的GC pause的时序图是基本一样的，只不过那里的GC pause发生在客户端1获得了锁之后，而这里的GC  pause发生在客户端1获得锁之前。但两个例子的侧重点不太一样。Martin构造这里的这个例子，是为了强调在一个分布式的异步环境下，长时间的GC pause或消息延迟（上面这个例子中，把GC  pause换成Redis节点和客户端1之间的消息延迟，逻辑不变），会让客户端获得一个已经过期的锁。从客户端1的角度看，Redlock的安全性被打破了，因为客户端1收到锁的时候，这个锁已经失效了，而Redlock同时还把这个锁分配给了客户端2。换句话说，Redis服务器在把锁分发给客户端的途中，锁就过期了，但又没有有效的机制让客户端明确知道这个问题。而在之前的那个例子中，客户端1收到锁的时候锁还是有效的，锁服务本身的安全性可以认为没有被打破，后面虽然也出了问题，但问题是出在客户端1和共享资源服务器之间的交互上。

在Martin的这篇文章中，还有一个很有见地的观点，就是对锁的用途的区分。他把锁的用途分为两种：

- 为了效率(efficiency)，协调各个客户端避免做重复的工作。即使锁偶尔失效了，只是可能把某些操作多做一遍而已，不会产生其它的不良后果。比如重复发送了一封同样的email。
- 为了正确性(correctness)。在任何情况下都不允许锁失效的情况发生，因为一旦发生，就可能意味着数据不一致(inconsistency)，数据丢失，文件损坏，或者其它严重的问题。

最后，Martin得出了如下的结论：

- 如果是为了效率(efficiency)而使用分布式锁，允许锁的偶尔失效，那么使用单Redis节点的锁方案就足够了，简单而且效率高。Redlock则是个过重的实现(heavyweight)。
- 如果是为了正确性(correctness)在很严肃的场合使用分布式锁，那么不要使用Redlock。它不是建立在异步模型上的一个足够强的算法，它对于系统模型的假设中包含很多危险的成分(对于timing)。而且，它没有一个机制能够提供fencing token。那应该使用什么技术呢？Martin认为，应该考虑类似Zookeeper的方案，或者支持事务的数据库。

Martin对Redlock算法的形容是：

> neither fish nor fowl （非驴非马）

【***其它疑问\***】

- Martin提出的fencing token的方案，需要对提供共享资源的服务进行修改，这在现实中可行吗？
- 根据Martin的说法，看起来，如果资源服务器实现了fencing token，它在分布式锁失效的情况下也仍然能保持资源的互斥访问。这是不是意味着分布式锁根本没有存在的意义了？
- 资源服务器需要检查fencing token的大小，如果提供资源访问的服务也是包含多个节点的（分布式的），那么这里怎么检查才能保证fencing token在多个节点上是递增的呢？
- Martin对于fencing token的举例中，两个fencing token到达资源服务器的顺序颠倒了（小的fencing  token后到了），这时资源服务器检查出了这一问题。如果客户端1和客户端2都发生了GC pause，两个fencing  token都延迟了，它们几乎同时达到了资源服务器，但保持了顺序，那么资源服务器是不是就检查不出问题了？这时对于资源的访问是不是就发生冲突了？
- 分布式锁+fencing的方案是绝对正确的吗？能证明吗？

### antirez的反驳





Martin在发表了那篇分析分布式锁的blog (How to do distributed locking)之后，该文章在Twitter和Hacker News上引发了广泛的讨论。但人们更想听到的是Redlock的作者antirez对此会发表什么样的看法。

Martin的那篇文章是在2016-02-08这一天发表的，但据Martin说，他在公开发表文章的一星期之前就把草稿发给了antirez进行review，而且他们之间通过email进行了讨论。不知道Martin有没有意料到，antirez对于此事的反应很快，就在Martin的文章发表出来的第二天，antirez就在他的博客上贴出了他对于此事的反驳文章，名字叫"Is Redlock safe?"，地址如下:

- http://antirez.com/news/101

这是高手之间的过招。antirez这篇文章也条例非常清晰，并且中间涉及到大量的细节。antirez认为，Martin的文章对于Redlock的批评可以概括为两个方面（与Martin文章的前后两部分对应）：

- 带有自动过期功能的分布式锁，必须提供某种fencing机制来保证对共享资源的真正的互斥保护。Redlock提供不了这样一种机制。
- Redlock构建在一个不够安全的系统模型之上。它对于系统的记时假设(timing assumption)有比较强的要求，而这些要求在现实的系统中是无法保证的。

antirez对这两方面分别进行了反驳。

首先，关于fencing机制。antirez对于Martin的这种论证方式提出了质疑：既然在锁失效的情况下已经存在一种fencing机制能继续保持资源的互斥访问了，那为什么还要使用一个分布式锁并且还要求它提供那么强的安全性保证呢？即使退一步讲，Redlock虽然提供不了Martin所讲的递增的fencing token，但利用Redlock产生的随机字符串(`my_random_value`)可以达到同样的效果。这个随机字符串虽然不是递增的，但却是唯一的，可以称之为unique token。antirez举了个例子，比如，你可以用它来实现“Check and Set”操作，原话是：

> When starting to work with a shared resource, we set its state to “`<token>`”, then we operate the read-modify-write only if the token is still the same when we write.（译文：当开始和共享资源交互的时候，我们将它的状态设置成“`<token>`”，然后仅在token没改变的情况下我们才执行“读取-修改-写回”操作。）

第一遍看到这个描述的时候，我个人是感觉没太看懂的。“Check and Set”应该就是我们平常听到过的CAS操作了，但它如何在这个场景下工作，antirez并没有展开说（在后面讲到Hacker News上的讨论的时候，我们还会提到）。

然后，antirez的反驳就集中在第二个方面上：关于算法在记时(timing)方面的模型假设。在我们前面分析Martin的文章时也提到过，Martin认为Redlock会失效的情况主要有三种：

- 时钟发生跳跃。
- 长时间的GC pause。
- 长时间的网络延迟。

antirez肯定意识到了这三种情况对Redlock最致命的其实是第一点：时钟发生跳跃。这种情况一旦发生，Redlock是没法正常工作的。而对于后两种情况来说，Redlock在当初设计的时候已经考虑到了，对它们引起的后果有一定的免疫力。所以，antirez接下来集中精力来说明通过恰当的运维，完全可以避免时钟发生大的跳动，而Redlock对于时钟的要求在现实系统中是完全可以满足的。

Martin在提到时钟跳跃的时候，举了两个可能造成时钟跳跃的具体例子：

- 系统管理员手动修改了时钟。
- 从NTP服务收到了一个大的时钟更新事件。

antirez反驳说：

- 手动修改时钟这种人为原因，不要那么做就是了。否则的话，如果有人手动修改Raft协议的持久化日志，那么就算是Raft协议它也没法正常工作了。
- 使用一个不会进行“跳跃”式调整系统时钟的ntpd程序（可能是通过恰当的配置），对于时钟的修改通过多次微小的调整来完成。

而Redlock对时钟的要求，并不需要完全精确，它只需要时钟差不多精确就可以了。比如，要记时5秒，但可能实际记了4.5秒，然后又记了5.5秒，有一定的误差。不过只要误差不超过一定范围，这对Redlock不会产生影响。antirez认为呢，像这样对时钟精度并不是很高的要求，在实际环境中是完全合理的。

好了，到此为止，如果你相信antirez这里关于时钟的论断，那么接下来antirez的分析就基本上顺理成章了。

关于Martin提到的能使Redlock失效的后两种情况，Martin在分析的时候恰好犯了一个错误（在[本文上半部分](http://mp.weixin.qq.com/s?__biz=MzA4NTg1MjM0Mg==&mid=2657261514&idx=1&sn=47b1a63f065347943341910dddbb785d&chksm=84479e13b3301705ea29c86f457ad74010eba8a8a5c12a7f54bcf264a4a8c9d6adecbe32ad0b&scene=21#wechat_redirect)已经提到过）。在Martin给出的那个由客户端GC pause引发Redlock失效的例子中，这个GC  pause引发的后果相当于在锁服务器和客户端之间发生了长时间的消息延迟。Redlock对于这个情况是能处理的。回想一下Redlock算法的具体过程，它使用起来的过程大体可以分成5步：

1. 获取当前时间。
2. 完成**获取锁**的整个过程（与N个Redis节点交互）。
3. 再次获取当前时间。
4. 把两个时间相减，计算**获取锁**的过程是否消耗了太长时间，导致锁已经过期了。如果没过期，
5. 客户端持有锁去访问共享资源。

在Martin举的例子中，GC  pause或网络延迟，实际发生在上述第1步和第3步之间。而不管在第1步和第3步之间由于什么原因（进程停顿或网络延迟等）导致了大的延迟出现，在第4步都能被检查出来，不会让客户端拿到一个它认为有效而实际却已经过期的锁。当然，这个检查依赖系统时钟没有大的跳跃。这也就是为什么antirez在前面要对时钟条件进行辩护的原因。

有人会说，在第3步之后，仍然可能会发生延迟啊。没错，antirez承认这一点，他对此有一段很有意思的论证，原话如下：

> The delay can only happen after steps 3, resulting into the lock to be  considered ok while actually expired, that is, we are back at the first  problem Martin identified of distributed locks where the client fails to stop working to the shared resource before the lock validity expires.  Let me tell again how this problem is common with *all the distributed locks implementations*, and how the token as a solution is both unrealistic and can be used with Redlock as well.（译文：延迟只能发生在第3步之后，这导致锁被认为是有效的而实际上已经过期了，也就是说，我们回到了Martin指出的第一个问题上，客户端没能够在锁的有效性过期之前完成与共享资源的交互。让我再次申明一下，这个问题对于*所有的分布式锁的实现*是普遍存在的，而且基于token的这种解决方案是不切实际的，但也能和Redlock一起用。）

这里antirez所说的“Martin指出的第一个问题”具体是什么呢？在[本文上半部分](http://mp.weixin.qq.com/s?__biz=MzA4NTg1MjM0Mg==&mid=2657261514&idx=1&sn=47b1a63f065347943341910dddbb785d&chksm=84479e13b3301705ea29c86f457ad74010eba8a8a5c12a7f54bcf264a4a8c9d6adecbe32ad0b&scene=21#wechat_redirect)我们提到过，Martin的文章分为两大部分，其中前半部分与Redlock没有直接关系，而是指出了任何一种带自动过期功能的分布式锁在没有提供fencing机制的前提下都有可能失效。这里antirez所说的就是指的Martin的文章的前半部分。换句话说，对于大延迟给Redlock带来的影响，恰好与Martin在文章的前半部分针对所有的分布式锁所做的分析是一致的，而这种影响不单单针对Redlock。Redlock的实现已经保证了它是和其它任何分布式锁的安全性是一样的。当然，与其它“更完美”的分布式锁相比，Redlock似乎提供不了Martin提出的那种递增的token，但antirez在前面已经分析过了，关于token的这种论证方式本身就是“不切实际”的，或者退一步讲，Redlock能提供的unique token也能够提供完全一样的效果。

另外，关于大延迟对Redlock的影响，antirez和Martin在Twitter上有下面的对话：

> **antirez**:@martinkl so I wonder if after my reply, we can at least agree about unbound messages delay to don’t cause any harm.
>
> **Martin**:@antirez Agree about message delay between app and lock server. Delay between  app and resource being accessed is still problematic.
>
> （译文：**antirez**问：我想知道，在我发文回复之后，我们能否在一点上达成一致，就是大的消息延迟不会给Redlock的运行造成损害。**Martin**答：对于客户端和锁服务器之间的消息延迟，我同意你的观点。但客户端和被访问资源之间的延迟还是有问题的。）

通过这段对话可以看出，对于Redlock在第4步所做的锁有效性的检查，Martin是予以肯定的。但他认为客户端和资源服务器之间的延迟还是会带来问题的。Martin在这里说的有点模糊。就像antirez前面分析的，客户端和资源服务器之间的延迟，对所有的分布式锁的实现都会带来影响，这不单单是Redlock的问题了。

以上就是antirez在blog中所说的主要内容。有一些点值得我们注意一下：

- antirez是同意大的系统时钟跳跃会造成Redlock失效的。在这一点上，他与Martin的观点的不同在于，他认为在实际系统中是可以避免大的时钟跳跃的。当然，这取决于基础设施和运维方式。
- antirez在设计Redlock的时候，是充分考虑了网络延迟和程序停顿所带来的影响的。但是，对于客户端和资源服务器之间的延迟（即发生在算法第3步之后的延迟），antirez是承认所有的分布式锁的实现，包括Redlock，是没有什么好办法来应对的。

讨论进行到这，Martin和antirez之间谁对谁错其实并不是那么重要了。只要我们能够对Redlock（或者其它分布式锁）所能提供的安全性的程度有充分的了解，那么我们就能做出自己的选择了。



### Hacker News上的一些讨论





针对Martin和antirez的两篇blog，很多技术人员在Hacker News上展开了激烈的讨论。这些讨论所在地址如下：

- 针对Martin的blog的讨论：https://news.ycombinator.com/item?id=11059738
- 针对antirez的blog的讨论：https://news.ycombinator.com/item?id=11065933

在Hacker News上，antirez积极参与了讨论，而Martin则始终置身事外。

下面我把这些讨论中一些有意思的点拿出来与大家一起分享一下（集中在对于fencing token机制的讨论上）。

关于antirez提出的“Check and Set”操作，他在blog里并没有详加说明。果然，在Hacker News上就有人出来问了。antirez给出的答复如下：

> You want to modify locked resource X. You set X.currlock = token. Then you  read, do whatever you want, and when you write, you "write-if-currlock  == token". If another client did X.currlock = somethingelse, the  transaction fails.

翻译一下可以这样理解：假设你要修改资源X，那么遵循下面的伪码所定义的步骤。

1. 先设置X.currlock = token。
2. 读出资源X（包括它的值和附带的X.currlock）。
3. 按照"write-if-currlock ==  token"的逻辑，修改资源X的值。意思是说，如果对X进行修改的时候，X.currlock仍然和当初设置进去的token相等，那么才进行修改；如果这时X.currlock已经是其它值了，那么说明有另外一方也在试图进行修改操作，那么放弃当前的修改，从而避免冲突。

随后Hacker News上一位叫viraptor的用户提出了异议，它给出了这样一个执行序列：

- A: X.currlock = Token_ID_A
- A: resource read
- A: is X.currlock still Token_ID_A? yes
- B: X.currlock = Token_ID_B
- B: resource read
- B: is X.currlock still Token_ID_B? yes
- B: resource write
- A: resource write

到了最后两步，两个客户端A和B同时进行写操作，冲突了。不过，这位用户应该是理解错了antirez给出的修改过程了。按照antirez的意思，判断X.currlock是否修改过和对资源的写操作，应该是一个原子操作。只有这样理解才能合乎逻辑，否则的话，这个过程就有严重的破绽。这也是为什么antirez之前会对fencing机制产生质疑：既然资源服务器本身都能提供互斥的原子操作了，为什么还需要一个分布式锁呢？因此，antirez认为这种fencing机制是很累赘的，他之所以还是提出了这种“Check and Set”操作，只是为了证明在提供fencing  token这一点上，Redlock也能做到。但是，这里仍然有一些不明确的地方，如果将"write-if-currlock ==  token"看做是原子操作的话，这个逻辑势必要在资源服务器上执行，那么第二步为什么还要“读出资源X”呢？除非这个“读出资源X”的操作也是在资源服务器上执行，它包含在“判断-写回”这个原子操作里面。而假如不这样理解的话，“读取-判断-写回”这三个操作都放在客户端执行，那么看不出它们如何才能实现原子性操作。在下面的讨论中，我们暂时忽略“读出资源X”这一步。

这个基于random token的“Check and Set”操作，如果与Martin提出的递增的fencing token对比一下的话，至少有两点不同：

- “Check and Set”对于写操作要分成两步来完成（设置token、判断-写回），而递增的fencing token机制只需要一步（带着token向资源服务器发起写请求）。
- 递增的fencing token机制能保证最终操作共享资源的顺序，那些延迟时间太长的操作就无法操作共享资源了。但是基于random token的“Check and Set”操作不会保证这个顺序，那些延迟时间太长的操作如果后到达了，它仍然有可能操作共享资源（当然是以互斥的方式）。

对于前一点不同，我们在后面的分析中会看到，如果资源服务器也是分布式的，那么使用递增的fencing token也要变成两步。

而对于后一点操作顺序上的不同，antirez认为这个顺序没有意义，关键是能互斥访问就行了。他写下了下面的话：

> So the goal is, when race conditions happen, to avoid them in some way.......Note also that when it happens that, because of delays, the clients are  accessing concurrently, the lock ID has little to do with the order in  which the operations were indented to happen.（译文： 我们的目标是，当竞争条件出现的时候，能够以**某种方式**避免。......还需要注意的是，当那种竞争条件出现的时候，比如由于延迟，客户端是同时来访问的，锁的ID的大小顺序跟那些操作真正想执行的顺序，是没有什么关系的。）

这里的lock ID，跟Martin说的递增的token是一回事。

随后，antirez举了一个“将名字加入列表”的操作的例子：

- T0: Client A receives new name to add from web.
- T0: Client B is idle
- T1: Client A is experiencing pauses.
- T1: Client B receives new name to add from web.
- T2: Client A is experiencing pauses.
- T2: Client B receives a lock with ID 1
- T3: Client A receives a lock with ID 2

你看，两个客户端（其实是Web服务器）执行“添加名字”的操作，A本来是排在B前面的，但获得锁的顺序却是B排在A前面。因此，antirez说，锁的ID的大小顺序跟那些操作真正想执行的顺序，是没有什么关系的。关键是能排出一个顺序来，能互斥访问就行了。那么，至于锁的ID是递增的，还是一个random token，自然就不那么重要了。

Martin提出的fencing  token机制，给人留下了无尽的疑惑。这主要是因为他对于这一机制的描述缺少太多的技术细节。从上面的讨论可以看出，antirez对于这一机制的看法是，它跟一个random token没有什么区别，而且，它需要资源服务器本身提供某种互斥机制，这几乎让分布式锁本身的存在失去了意义。围绕fencing  token的问题，还有两点是比较引人注目的，Hacker News上也有人提出了相关的疑问：

- （1）关于资源服务器本身的架构细节。
- （2）资源服务器对于fencing token进行检查的实现细节，比如是否需要提供一种原子操作。

关于上述问题（1），Hacker News上有一位叫dwenzek的用户发表了下面的评论：

> ...... the issue around the usage of fencing tokens to reject any late usage  of a lock is unclear just because the protected resource and its access  are themselves unspecified. Is the resource distributed or not? If  distributed, does the resource has a mean to ensure that tokens are  increasing over all the nodes? Does the resource have a mean to rollback any effects done by a client which session is interrupted by a timeout?
>
> （译文：...... 关于使用fencing  token拒绝掉延迟请求的相关议题，是不够清晰的，因为受保护的资源以及对它的访问方式本身是没有被明确定义过的。资源服务是不是分布式的呢？如果是，资源服务有没有一种方式能确保token在所有节点上递增呢？对于客户端的Session由于过期而被中断的情况，资源服务有办法将它的影响回滚吗？）

这些疑问在Hacker News上并没有人给出解答。而关于分布式的资源服务器架构如何处理fencing token，另外一名分布式系统的专家Flavio Junqueira在他的一篇blog中有所提及（我们后面会再提到）。

关于上述问题（2），Hacker News上有一位叫reza_n的用户发表了下面的疑问：

> I understand how a fencing token can prevent out of order writes when 2  clients get the same lock. But what happens when those writes happen to  arrive in order and you are doing a value modification? Don't you still  need to rely on some kind of value versioning or optimistic locking?  Wouldn't this make the use of a distributed lock unnecessary?
>
> （译文： 我理解当两个客户端同时获得锁的时候fencing token是如何防止乱序的。但是如果两个写操作恰好按序到达了，而且它们在对同一个值进行修改，那会发生什么呢？难道不会仍然是依赖某种数据版本号或者乐观锁的机制？这不会让分布式锁变得没有必要了吗？）

一位叫Terr_的Hacker News用户答：

> I believe the "first" write fails, because the token being passed in is  no longer "the lastest", which indicates their lock was already released or expired.
>
> （译文： 我认为“第一个”写请求会失败，因为它传入的token不再是“最新的”了，这意味着锁已经释放或者过期了。）

Terr_的回答到底对不对呢？这不好说，取决于资源服务器对于fencing token进行检查的实现细节。让我们来简单分析一下。

为了简单起见，我们假设有一台（先不考虑分布式的情况）通过RPC进行远程访问文件服务器，它无法提供对于文件的互斥访问（否则我们就不需要分布式锁了）。现在我们按照Martin给出的说法，加入fencing token的检查逻辑。由于Martin没有描述具体细节，我们猜测至少有两种可能。

第一种可能，我们修改了文件服务器的代码，让它能多接受一个fencing token的参数，并在进行所有处理之前加入了一个简单的判断逻辑，保证只有当前接收到的fencing token大于之前的值才允许进行后边的访问。而一旦通过了这个判断，后面的处理不变。

现在想象reza_n描述的场景，客户端1和客户端2都发生了GC pause，两个fencing  token都延迟了，它们几乎同时到达了文件服务器，而且保持了顺序。那么，我们新加入的判断逻辑，应该对两个请求都会放过，而放过之后它们几乎同时在操作文件，还是冲突了。既然Martin宣称fencing token能保证分布式锁的正确性，那么上面这种可能的猜测也许是我们理解错了。

当然，还有第二种可能，就是我们对文件服务器确实做了比较大的改动，让这里判断token的逻辑和随后对文件的处理放在一个原子操作里了。这可能更接近antirez的理解。这样的话，前面reza_n描述的场景中，两个写操作都应该成功。



### 基于ZooKeeper的分布式锁更安全吗？

Zookeeper是通过创建临时顺序节点的方式来实现。

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/ibBMVuDfkZUkr0NZvdnibzGXkPiaucEtNLZgr0e3ythcYHu2QkEk1C4icaibSfHCluFrzwLTmG0TsVTLeRl7hAm25kA/640?wx_fmt=jpeg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

1. 当需要对资源进行加锁时，实际上就是在父节点之下创建一个临时顺序节点。
2. 客户端A来对资源加锁，首先判断当前创建的节点是否为最小节点，如果是，那么加锁成功，后续加锁线程阻塞等待
3. 此时，客户端B也来尝试加锁，由于客户端A已经加锁成功，所以客户端B发现自己的节点并不是最小节点，就会去取到上一个节点，并且对上一节点注册监听
4. 当客户端A操作完成，释放锁的操作就是删除这个节点，这样就可以触发监听事件，客户端B就会得到通知，同样，客户端B判断自己是否为最小节点，如果是，那么则加锁成功



很多人（也包括Martin在内）都认为，如果你想构建一个更安全的分布式锁，那么应该使用ZooKeeper，而不是Redis。那么，为了对比的目的，让我们先暂时脱离开本文的题目，讨论一下基于ZooKeeper的分布式锁能提供绝对的安全吗？它需要fencing token机制的保护吗？

我们不得不提一下分布式专家Flavio Junqueira所写的一篇blog，题目叫“Note on fencing and distributed locks”，地址如下：

- https://fpj.me/2016/02/10/note-on-fencing-and-distributed-locks/

Flavio Junqueira是ZooKeeper的作者之一，他的这篇blog就写在Martin和antirez发生争论的那几天。他在文中给出了一个基于ZooKeeper构建分布式锁的描述（当然这不是唯一的方式）：

- 客户端尝试创建一个znode节点，比如`/lock`。那么第一个客户端就创建成功了，相当于拿到了锁；而其它的客户端会创建失败（znode已存在），获取锁失败。
- 持有锁的客户端访问共享资源完成后，将znode删掉，这样其它客户端接下来就能来获取锁了。
- znode应该被创建成ephemeral的。这是znode的一个特性，它保证如果创建znode的那个客户端崩溃了，那么相应的znode会被自动删除。这保证了锁一定会被释放。

看起来这个锁相当完美，没有Redlock过期时间的问题，而且能在需要的时候让锁自动释放。但仔细考察的话，并不尽然。

ZooKeeper是怎么检测出某个客户端已经崩溃了呢？实际上，每个客户端都与ZooKeeper的某台服务器维护着一个Session，这个Session依赖定期的心跳(heartbeat)来维持。如果ZooKeeper长时间收不到客户端的心跳（这个时间称为Sesion的过期时间），那么它就认为Session过期了，通过这个Session所创建的所有的ephemeral类型的znode节点都会被自动删除。

设想如下的执行序列：

1. 客户端1创建了znode节点`/lock`，获得了锁。
2. 客户端1进入了长时间的GC pause。
3. 客户端1连接到ZooKeeper的Session过期了。znode节点`/lock`被自动删除。
4. 客户端2创建了znode节点`/lock`，从而获得了锁。
5. 客户端1从GC pause中恢复过来，它仍然认为自己持有锁。

最后，客户端1和客户端2都认为自己持有了锁，冲突了。这与之前Martin在文章中描述的由于GC pause导致的分布式锁失效的情况类似。

看起来，用ZooKeeper实现的分布式锁也不一定就是安全的。该有的问题它还是有。但是，ZooKeeper作为一个专门为分布式应用提供方案的框架，它提供了一些非常好的特性，是Redis之类的方案所没有的。像前面提到的ephemeral类型的znode自动删除的功能就是一个例子。

还有一个很有用的特性是ZooKeeper的watch机制。这个机制可以这样来使用，比如当客户端试图创建`/lock`的时候，发现它已经存在了，这时候创建失败，但客户端不一定就此对外宣告获取锁失败。客户端可以进入一种等待状态，等待当`/lock`节点被删除的时候，ZooKeeper通过watch机制通知它，这样它就可以继续完成创建操作（获取锁）。这可以让分布式锁在客户端用起来就像一个本地的锁一样：加锁失败就阻塞住，直到获取到锁为止。这样的特性Redlock就无法实现。

小结一下，基于ZooKeeper的锁和基于Redis的锁相比在实现特性上有两个不同：

- 在正常情况下，客户端可以持有锁任意长的时间，这可以确保它做完所有需要的资源访问操作之后再释放锁。这避免了基于Redis的锁对于有效时间(lock validity  time)到底设置多长的两难问题。实际上，基于ZooKeeper的锁是依靠Session（心跳）来维持锁的持有状态的，而Redis不支持Sesion。
- 基于ZooKeeper的锁支持在获取锁失败之后等待锁重新释放的事件。这让客户端对锁的使用更加灵活。

顺便提一下，如上所述的基于ZooKeeper的分布式锁的实现，并不是最优的。它会引发“herd effect”（羊群效应），降低获取锁的性能。一个更好的实现参见下面链接：

- http://zookeeper.apache.org/doc/r3.4.9/recipes.html#sc_recipes_Locks

我们重新回到Flavio Junqueira对于fencing token的分析。Flavio Junqueira指出，fencing  token机制本质上是要求客户端在每次访问一个共享资源的时候，在执行任何操作之前，先对资源进行某种形式的“标记”(mark)操作，这个“标记”能保证持有旧的锁的客户端请求（如果延迟到达了）无法操作资源。这种标记操作可以是很多形式，fencing token是其中比较典型的一个。

随后Flavio Junqueira提到用递增的epoch number（相当于Martin的fencing  token）来保护共享资源。而对于分布式的资源，为了方便讨论，假设分布式资源是一个小型的多备份的数据存储(a small replicated  data store)，执行写操作的时候需要向所有节点上写数据。最简单的做标记的方式，就是在对资源进行任何操作之前，先把epoch  number标记到各个资源节点上去。这样，各个节点就保证了旧的（也就是小的）epoch number无法操作数据。

当然，这里再展开讨论下去可能就涉及到了这个数据存储服务的实现细节了。比如在实际系统中，可能为了容错，只要上面讲的标记和写入操作在多数节点上完成就算成功完成了（Flavio  Junqueira并没有展开去讲）。在这里我们能看到的，最重要的，是这种标记操作如何起作用的方式。这有点类似于Paxos协议（Paxos协议要求每个proposal对应一个递增的数字，执行accept请求之前先执行prepare请求）。antirez提出的random token的方式显然不符合Flavio  Junqueira对于“标记”操作的定义，因为它无法区分新的token和旧的token。只有递增的数字才能确保最终收敛到最新的操作结果上。

在这个分布式数据存储服务（共享资源）的例子中，客户端在标记完成之后执行写入操作的时候，存储服务的节点需要判断epoch  number是不是最新，然后确定能不能执行写入操作。如果按照上一节我们的分析思路，这里的epoch判断和接下来的写入操作，是不是在一个原子操作里呢？根据Flavio  Junqueira的相关描述，我们相信，应该是原子的。那么既然资源本身可以提供原子互斥操作了，那么分布式锁还有存在的意义吗？应该说有。客户端可以利用分布式锁有效地避免冲突，等待写入机会，这对于包含多个节点的分布式资源尤其有用（当然，是出于效率的原因）。



### Chubby的分布式锁是怎样做fencing的？





提到分布式锁，就不能不提Google的Chubby。

Chubby是Google内部使用的分布式锁服务，有点类似于ZooKeeper，但也存在很多差异。Chubby对外公开的资料，主要是一篇论文，叫做“The Chubby lock service for loosely-coupled distributed systems”，下载地址如下：

- https://research.google.com/archive/chubby.html

另外，YouTube上有一个的讲Chubby的talk，也很不错，播放地址：

- https://www.youtube.com/watch?v=PqItueBaiRg&feature=youtu.be&t=487

Chubby自然也考虑到了延迟造成的锁失效的问题。论文里有一段描述如下：

> a process holding a lock L may issue a request R, but then fail. Another  process may ac- quire L and perform some action before R arrives at its  destination. If R later arrives, it may be acted on without the  protection of L, and potentially on inconsistent data.
>
> （译文： 一个进程持有锁L，发起了请求R，但是请求失败了。另一个进程获得了锁L并在请求R到达目的方之前执行了一些动作。如果后来请求R到达了，它就有可能在没有锁L保护的情况下进行操作，带来数据不一致的潜在风险。）

这跟Martin的分析大同小异。

Chubby给出的用于解决（缓解）这一问题的机制称为sequencer，类似于fencing token机制。锁的持有者可以随时请求一个sequencer，这是一个字节串，它由三部分组成：

- 锁的名字。
- 锁的获取模式（排他锁还是共享锁）。
- lock generation number（一个64bit的单调递增数字）。作用相当于fencing token或epoch number。

客户端拿到sequencer之后，在操作资源的时候把它传给资源服务器。然后，资源服务器负责对sequencer的有效性进行检查。检查可以有两种方式：

- 调用Chubby提供的API，CheckSequencer()，将整个sequencer传进去进行检查。这个检查是为了保证客户端持有的锁在进行资源访问的时候仍然有效。
- 将客户端传来的sequencer与资源服务器当前观察到的最新的sequencer进行对比检查。可以理解为与Martin描述的对于fencing token的检查类似。

当然，如果由于兼容的原因，资源服务本身不容易修改，那么Chubby还提供了一种机制：

- lock-delay。Chubby允许客户端为持有的锁指定一个lock-delay的时间值（默认是1分钟）。当Chubby发现客户端被动失去联系的时候，并不会立即释放锁，而是会在lock-delay指定的时间内阻止其它客户端获得这个锁。这是为了在把锁分配给新的客户端之前，让之前持有锁的客户端有充分的时间把请求队列排空(draining the queue)，尽量防止出现延迟到达的未处理请求。

可见，为了应对锁失效问题，Chubby提供的三种处理方式：CheckSequencer()检查、与上次最新的sequencer对比、lock-delay，它们对于安全性的保证是从强到弱的。而且，这些处理方式本身都没有保证提供绝对的正确性(correctness)。但是，Chubby确实提供了单调递增的lock generation number，这就允许资源服务器在需要的时候，利用它提供更强的安全性保障。



### 关于时钟



在Martin与antirez的这场争论中，冲突最为严重的就是对于系统时钟的假设是不是合理的问题。Martin认为系统时钟难免会发生跳跃（这与分布式算法的异步模型相符），而antirez认为在实际中系统时钟可以保证不发生大的跳跃。

Martin对于这一分歧发表了如下看法（原话）：

> So, fundamentally, this discussion boils down to whether it is reasonable  to make timing assumptions for ensuring safety properties. I say no,  Salvatore says yes — but that's ok. Engineering discussions rarely have  one right answer.
>
> （译文：从根本上来说，这场讨论最后归结到了一个问题上：为了确保安全性而做出的记时假设到底是否合理。我认为不合理，而antirez认为合理 —— 但是这也没关系。工程问题的讨论很少只有一个正确答案。）

那么，在实际系统中，时钟到底是否可信呢？对此，Julia Evans专门写了一篇文章，“TIL: clock skew exists”，总结了很多跟时钟偏移有关的实际资料，并进行了分析。这篇文章地址：



- http://jvns.ca/blog/2016/02/09/til-clock-skew-exists/



Julia Evans在文章最后得出的结论是：

> clock skew is real（时钟偏移在现实中是存在的）



### Martin的事后总结





我们前面提到过，当各方的争论在激烈进行的时候，Martin几乎始终置身事外。但是Martin在这件事过去之后，把这个事件的前后经过总结成了一个很长的故事线。如果你想最全面地了解这个事件发生的前后经过，那么建议去读读Martin的这个总结：

- https://storify.com/martinkl/redlock-discussion

在这个故事总结的最后，Martin写下了很多感性的评论：

> For me, this is the most important point: I don't care who is right or  wrong in this debate — I care about learning from others' work, so that  we can avoid repeating old mistakes, and make things better in future.  So much great work has already been done for us: by standing on the  shoulders of giants, we can build better software.......By all means, test ideas by arguing them and checking whether they stand  up to scrutiny by others. That's part of the learning process. But the  goal should be to learn, not to convince others that you are right.  Sometimes that just means to stop and think for a while.
>
> （译文：对我来说最重要的一点在于：我并不在乎在这场辩论中谁对谁错 —— 我只关心从其他人的工作中学到的东西，以便我们能够避免重蹈覆辙，并让未来更加美好。前人已经为我们创造出了许多伟大的成果：站在巨人的肩膀上，我们得以构建更棒的软件。......对于任何想法，务必要详加检验，通过论证以及检查它们是否经得住别人的详细审查。那是学习过程的一部分。但目标应该是为了获得知识，而不应该是为了说服别人相信你自己是对的。有时候，那只不过意味着停下来，好好地想一想。）



------