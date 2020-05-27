http://www.redis.cn/documentation.html

#### Redis 支持的五种类型：

##### 1）string：

​		二进制安全（可以用于保存图片等二进制文件），set key value、get key，常规的字符串操作、数值操作、位操作

##### 2）hash：

​		理解为 java 中的 map，hset key filed value、hget key field、hgetall key，保存对象

##### 3）list：

​		双端链表实现的列表，并且保存链表长度。lpush key value、lpop key、rpush key value、rpop key，实现表、栈、队列

##### 4）set：

​		无序集合，sadd key member、sismember key member，交并差集

​    交并差集

##### 5）sorted set：

​		排序集合，跳跃表实现，可以指定一个双精度数作为序号，相同序号以字典序排序，zadd key score member

以下为一些导出类型

##### 6）bitmap

​		利用 string 类型的二进制安全的特性，可以进行一些位运算。最长为 2^32 个位，512MB，4E+。

##### 7）HyperLogLog

​		基础类型为 string。在做一些近似统计时，可以使用这种数据结构（算法）。花费 12kb  的内存，就可以对大量数据进行统计，误差在1%左右。这个思路来自于抛硬币实验，连续正面和抛的次数的关系。

​		pfadd 值的时候，先做 hash，低14位作为桶，从第15位开始数连续的0，保存做多连续的0的个数。pfcount 时取调和平均数 * 桶数 * lgM。合并时对应的桶取较大值。

##### 8）地理空间

​		将给定的经纬度通过 geohash（经纬度不停地折半查找，上半取1下半取0，得到两个二进制数，然后按位交叉排列） 处理，表示成一个 52 位的整数，保存在 sorted set 的score 里面，从而可以计算两个位置的距离，也可以查找某个位置附近的位置。



#### Redis 运行原理：

##### 1）线程模型：

​    工作线程为单线程模型，IO 线程为多路复用模型。

​    通过多路复用机制，利用 linux 的 epoll 功能同时监听多个 socket，当有信息消息过来时，消息会被封装成事件放入一个队列中。工作线程从这个队列中依次读取、处理事件。如果 delete 一个很大的 key，可能会在另一个线程中进行处理。

##### 2）架构模型：

a）单点模型：

​    一台机器一个进程。问题：①数据量变大之后性能下降。②单点故障。

b）集群模型：

​    针对性能下降问题作的改进。数据量分配到不同的机器上，使每台机器只需要承担1/n的数据量和流量。redis 集群分为 1024 *2 * 8 = 16384 个插槽，每个节点负责一部分数据。问题：如果 key 跨机器是不能联合操作的。

c）主从集群：

​    针对单点故障和读性能的问题。配置一台主节点和多台从节点，所有写操作打到主节点，读操作可以打到任意节点。从节点通过主节点的操作日志更新自己的数据。这里涉及到一个数据一致性的问题。如果主节点等待所有从节点数据更新成功才返回，则叫做强一致性，性能较低，如果某个从节点失效，甚至可能导致系统崩溃。主节点也可以直接返回，而去异步地更新从节点的数据，这种方式叫弱一致性，性能较高，但是数据可能不一致。redis 默认工作在弱一致性的模式下。主节点如果失效，可以手动把一个从节点切换为主节点，并且更改其他从节点的配置和客户端的配置。

d）哨兵集群：

​    在主从集群的基础上，增加一个哨兵集群。哨兵集群本身是一个基于 raft 算法的分布式系统，一个采用基数个节点实现。哨兵的初始配置只需要指定 redis 的主节点即可。哨兵通过 ping 命令确认其他哨兵和 redis 实例是否存活。哨兵通过 发布/订阅 和其他哨兵交换信息。哨兵通过 info 命令获取 redis 实例的信息。

​    当主节点出现故障时，每个哨兵发现 ping 失败之后，判断主节点为主观下线，并发布这个消息。如果某个哨兵根据自己的 ping 命令的结果和其他哨兵的消息发现有超过一半的哨兵判断主节点已经主观下线，则判断主节点为客观下线。这个时候需要进行故障转移操作。

​    发现主节点客观下线的哨兵发起投票，请求成为 leader 哨兵，其他哨兵进行投票，但每个哨兵只能投一票。如果超时仍然没有投出 leader，则会再次进行投票。若干轮后选出一个 leader。然后 leader 通过一些算法（数据最新、最近通信、id 号等），选出最佳节点成为主节点，然后更新集群内各个节点的信息，最后通知其他哨兵，使用新的主节点。

##### 3）内存模型

​		通过 info memory 查看。

​		used_memory：分配器分配的内存总量

​		used_memory_rss：进程占的总内存，和操作系统 top 和 ps 看到的一致。包括分配内存、进程内存、内存碎片

​		mem_fragmentation_ratio：内存碎片比例

​		mem_allocator：内存分配器，默认 jemalloc，其他有 libc，tcmalloc

##### 4）通讯协议 RESP（REdis Serialization Protocol）：

​		Redis 使用一种自定义的协议进行 TCP 通信，考虑到简单实现、快速解析和人工已读的特点。

​		单行回复：+*\r\n

​		错误消息：-*\r\n

​		整型数字：:*\r\n

​		批量回复：$\r\n*\r\n，$-1

​		多个批量回复：*个数\r\n$字节数\r\n内容\r\n$-1



#### 持久化

##### 1）RDB

​		通过某些条件触发生成当前数据快照，主进程 fork 出一个子进程去执行这个任务。save和bgsave命令；配置文件save m n规则；从节点全量复制；flushall 清空；shutdown 关闭

##### 2）AOF

​		追加每一条写命令到日志中。先把写命令写到缓冲区，然后通过操作系统、每条或定时的方式去刷新磁盘。在调用 bgrewriteaof 命令或 auto-aof-rewrite-min-size、auto-aof-rewrite-percentage 参数被满足后，会进行简单的日志合并。

##### 3）混合持久化

​		综合以上两种方式，通过 aof-use-rdb-preamble 参数开启。通过 bgrewriteaof  命令，先生成快照，在追加日志。



#### 应用

##### 1）分布式锁

​		在不是很大的系统中，redis 可能就一台主节点，利用其单工作线程的特点，来完成原子化的操作。

​		基本思路：通过类似 setnx 命令，在不存在 key 的情况下，设置一个 value。这里的 key 就是锁，value 就是持有锁的对象，通过这个命令的返回值判断获取锁是否成功，通过 value 来防止未获取到锁的对象进行解锁操作。一般还要设置一个过期时间，防止某个对象长期持有锁。

##### 2）唯一主键生成器

​		也是利用其单线程的工作模型，可以对一个数值型的值进行自增操作，从而获取到下一个唯一 id。如果这个操作比较频繁，可以一次增加100或1000，并把这一块 id 分配给同一个客户端，再由客户端去分配每一个 id。

##### 3）登录频率

​		利用其位运算的方便性，用一位数据表示登录与否之类的布尔类型的数据，方便操作和统计，而且节省空间。

##### 4）内存数据库

​		可以用来保存一些数据，通过持久化技术使数据在 redis 重启之后依然不丢失。这个场景谨慎使用，因为从 redis 的设计角度来说，它追求的是高性能，所以数据可能出现不一致的情况。如果开启强一致性，那 redis 就失去了它的优势。

##### 5）缓存

​		应该是 redis 的最主要的用途。

​		缓存雪崩：大面积的缓存失效。通过算法使缓存失效时间随机分布。

​		缓存穿透：查询不存在的数据。布隆过滤器



#### 淘汰策略

##### 1）过期删除

​		通过 expire 命令可以设置一个 key 的过期时间。一个 key 过期之后，redis 不会立即删除，而是采用以下两种策略进行删除。

​		惰性删除：当一个过期的 key 被访问的时候，出发删除。

​		定期删除：没 0.1s 随机测试 20 个 key，删除过期的 key；如果过期的 key 达到 25%，则重复以上流程

##### 2）内存回收		

​		达到 maxmemory 的限制时触发内存回收，maxmemory-policy 参数可以指定以下几个值：

​		noeviction：不回收，直接返回错误

​		allkeys-lru：回收最少使用的键

​		volatile-lru：回收过期的最少使用的键

​		allkeys-random：随机回收

​		volatile-random：随机回收过期的键

​		volatile-ttl：回收过期的键，优先回收存活时间较短的键

##### 3）从节点过期策略

​		从节点不会使 key 过期，而是等待主节点发送 del 命令来删除过期的 key。同时从节点使用一个逻辑时钟来避免返回过期数据，从而避免主节点 del 命令延迟导致的数据不一致。



#### SDS（Simple Dynamic String）数据

​		Redis 定义的一种存数据的结构体，保存一个 char 数组，一个 len 长度，一个 free 未使用字节



#### Redis 中的命令：

##### 1）keys 相关：

​		keys pattern：返回匹配的 key

##### 2）大量数据操作：

​		scan（key），sscan（set），hscan（hash），zscan（sorted set）：第一个不需要 key，后三个需要 key，可以指定正则表达式和一个数量。第一次传入0，后面每一个传入上一次返回的值。

##### 3）strings 相关：（NX = not exist， XX = exist）

​	append key value：字符串拼接，返回拼接后字符串的长度

​    decr key：数值减1

​    decrby key decrement：数值减 decrement

​    incr key：数值加1

​    incrby key increment：数值加 increment

​    incrbyfloat key increment：浮点数相加

​    get key：返回字符串

​    mget key [key ...]：批量返回字符串

​    set key value [s] [ms] [nx|xx]：设置一个字符串，同时可以设置过期时间和在key存在与否的情况进行设置

​    mset key value [key value ...]：原子性地批量设置字符串

​    msetnx key value [key value ...]：同上，但所有key必须不存在

​    getset key value：设置 value 并返回之前的 value

​    getrange key start end：返回指定字节范围内的子字符串

​    setex key s value：设置一个有过期时间的字符串

​    psetex key ms value：同setex

​    setnx key value：在key不存在时设置字符串

​    bitcount key [start end]：统计指定字节范围内位为1的数量

​    setrange key offset value：在指定的字节位置处设置字符串

​    bitop [and|or|not|xor] destkey key [key ...]：对指定的 key 的数据进行按位操作，结果保存在 destkey 中

​    bitpos key [0|1] [start end]：返回指定字节范围内第一个被设置为0或1的位的位置，没有则返回-1

​    getbit key offset：返回指定bit的值

​    setbit key offset [0|1]：对某一位置0或1

​    strlen key：返回字符串的字节长度