Redis基础数据结构

1. String:简单的key-value结构，即一个key对应一个string字符串，一个键最大能够存储512MB数据
2. Hash: Hash类型与JAVA的Map，一个key对应一个Map组,即key-[field,value]这样的数据结构
3. List : redis的list是简单的字符串列表，底层数据结构是链表，排序顺序是插入顺序，可以选择从链表头部或者尾部插入数据，查询是可以根据范围查询，结构为key-[value1,value2....valueN]
4. Set :  set是string类型的无序集合，通过哈希实现，所以各种查询插入，删除操作时间复杂度都是O(1)不能够存入重复数据，结构为：key-[value1,value2....valueN]，个人理解与list类似，但无序
5. Zset(SortedSet) ：Zset与普通set类型，不允许重复数据出现，也是哈希实现，各种操作 复杂度都是O(1)，但是Zset的每条数据都会关联一个double类型的分数字段，redis通过该字段对zset数据进行排序，double类型的分数可以重复

Redis高级数据结构

1. HyperLogLog
2. Geo
3. Pub/Sub
4. BloomFilter(布隆过滤器) ：布隆过滤器，通常用于检索一个元素是否存在于一个集合中，通常为了解决业务场景需要判断是否存在相同数据，但是做出该判断需要遍历检索大量已存在数据，以此浪费了大量的时间，布隆过滤器就是为了解决这个问题，实现原理主要是通过对数据的‘值’进行哈希计算，然后将值放入到对应位置，如果需要判断该值是否存在，只需要对数据进行哈希处理后检索对应索引位置即可。**布隆过滤器的缺点**即误判率，通常布隆过滤器检查值是否存在时会有两种结果：可能存在集合中、绝对不存在集合中，因为不同的值经过多次哈希计算后，可能会出现相同的结果，即有可能两个值不同，但是数据相同，所以出现了布隆过滤器判断数据‘可能存在集合中‘的情况
5. Redis Module
6. Redis Search
7. Redis-ML

个人对redis数据结构Zset底层结构——跳跃表的理解

```
跳跃表，根据相关文档介绍，是一种可以与平衡树聘美的层次化链表结构——删除、查找、添加等操作，都可以在期望时间下完成。
跳跃表的本质是为了解决查找数据速度慢的问题，Zset的所有节点都存在一个score值，Zset本身也按照这个值排序，通常链表插入新节点，需要将节点插入到指定的位置，以此来保证链表是有序的，定位插入点通常会使用二分查找法，但是二分查找是查找有序数组的，链表没有办法进行定位，所以二分查找法其实不是一个好的解决办法
如果在普通链表的两个相邻节点都添加一个指针，让指针指向下一组相邻的节点的指针，如下图
```

![image-20210207165634334](C:\Users\p\AppData\Roaming\Typora\typora-user-images\image-20210207165634334.png)

```
这样，如果我们现在想要查找分数为7的节点，只需要从节点3->11<-7就能够找到目标节点，即先找到3，再找到7，发现该层级分数已经大于目标节点分数，现在就需要回退到下一层级的节点，然后就找到了分数为7的节点，这样在查找数据时能够想象到，可以减少遍历节点数据的次数，根据这个例子，我们可以继续改进，如果数据够多，我们就添加更多的节点层级，以此减少节点的遍历，提升查找数据的速度
```

```
跳跃表 skiplist，跳跃表就是受刚才的多层链表结构启发而设计出来的，上面的表结构虽然比较好，但是如果有新元素插入，或者删除了某个节点元素，则需要重新对链表层级关系进行维护，不得不删除目标节点后面的数据然后重新插入，但这样又会带来更多的性能开销，跳跃表本身对这种操作进行了优化
跳跃表对每个节点的层级选择进行了单独的计算，每个节点在新增时都会出现一个随机层数，如下图
```

![图片](https://mmbiz.qpic.cn/mmbiz_png/ia1kbU3RS1H5ZLiaicqeR9mzkQuQLwvtFfQ5qUqf8c0vC3bfbc710Tz6iadcOlDYb39pApOUP9pCaUDQtuicUn9Jibvg/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

```
这样可以看出来，每个节点的插入只需要修改目标位置的前后节点指向，不用对多个节点进行操作，降低了插入所需要执行的操作，保证了执行完成的时间。
跳跃表的随机层数，由一个随机算法生成，直观上期望的目标是 50% 的概率被分配到 Level 1，25% 的概率被分配到 Level 2，12.5% 的概率被分配到 Level 3，以此类推...有 2-63 的概率被分配到最顶层，因为这里每一层的晋升率都是 50%。
同时跳跃表的最大允许层数为32层，当 Level[0] 有 264 个元素时，才能达到 32 层，所以定义 32 完全够用了。
因此跳跃表能够实现在很快的速度中实现精准的定位，然后进行查找、删除、新增等操作
```

Redis的缓存雪崩、击穿、穿透

1. 缓存雪崩
   - 一般出现缓存雪崩的情况，都是由于大量的缓存失效，由于设置的缓存有效时间几乎一致，大量缓存在同一时间同时失效，并且失效的同时系统有活动导致用户请求大量涌入，大量的请求没有经过缓存直接访问了数据库服务器，导致服务宕机。
   - 解决办法一般是设置缓存的失效时间为随机数，让缓存的有效期能够均匀遍布在每个时间点，保证缓存的有效使用。
2. 缓存穿透
   - 缓存穿透一般是指缓存和数据库中都不存在的数据被持续请求，系统在缓存中找不到该数据，只能查询数据库，但数据库中也不存在，就只能不断查询，发起请求，例如数据id为自增时，不断查询id为-1的数据。这种情况会添加数据库的压力，甚至击垮数据库。
   - 解决办法：缓存穿透这种情况一般需要在接口部分做参数校验和用户鉴权，对不合法的参数请求做拦截操作即可。
   - 使用布隆过滤器也是一个不错的选择，布隆过滤器能够替数据库判断该数据是否存在于集合中，如果不存在则直接拦截请求
3. 缓存击穿
   - 这里举个例子，例如微博突然出现了xx的热搜新闻，导致大量请求访问该热搜，但是由于这种情况持续时间久并且访问量较大，在这个热搜缓存过期的一瞬间，大量的用户请求都直接访问了数据库，导致数据库直接宕机这样的情况。
   - 解决办法：针对缓存击穿这种情况，只需要设置热点数据永不过期即可。

Redis为什么那么快：

1. 完全基于内存，redis的绝大多数请求都是内存操作，它的数据存在于内存中，数据的查找和操作时间复杂度都是O(1)
2. 数据结构简单，并且redis的数据结构都为此设计并且优化
3. 单线程设计，避免了必要的上下文切换和竞争条件，不存在各种锁操作，没有多余的性能消耗
4. 使用多路I/O复用模型，非阻塞IO
5. 使用底层模型不同，他们redis自己构建了VM机制，因为使用一般的系统调用函数会浪费一定的时间去移动和请求

什么是上下文切换：

- 当前线程处理数据，遇到问题，需要切换线程，此时切换线程的流程叫做上下文切换

redis服务是单线程，但现在服务器都是多核的，redis会不会很浪费？

- 可以通过启动多个redis实例重复使用系统资源

redis集群
	- 主从复制：一个主库从库，读写分离分担压力，主库和从库宕机都会导致请求失败，数据可能不一致的情况
	- 哨兵模式：引入哨兵来监控和处理故障，主库出现故障可以选举从库为主库
	- Cluster模式：解决ridis容量受限于单机
