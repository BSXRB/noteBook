# redis启动

1. 根据指定配置文件启动

`	[root@master local]# redis/bin/redis-server ./lib/redis-5.0.3/redis.conf`

`	[root@master local]# redis/bin/redis-cli`

若更改了端口使用redis-cli 连接客户端时也需要指定端口

`	[root@master local]# redis/bin/redis-cli -p 6380`

输入密码：

`	127.0.0.1:6379> auth llc782923640+`

2. 使用redis启动脚本设置开机自动启动

   启动脚本在redis安装目录/utils/redis_init_stcript：

   ```bash
   #!/bin/sh
   #
   # Simple Redis init.d script conceived to work on Linux systems
   # as it does use of the /proc filesystem.
    
   #redis服务器监听的端口
   REDISPORT=6379
    
   #服务端所处位置
   EXEC=/usr/local/bin/redis-server
    
   #客户端位置
   CLIEXEC=/usr/local/bin/redis-cli
    
   #redis的PID文件位置，需要修改
   PIDFILE=/var/run/redis_${REDISPORT}.pid
    
   #redis的配置文件位置，需将${REDISPORT}修改为文件名
   CONF="/etc/redis/${REDISPORT}.conf"
    
   case "$1" in
       start)
           if [ -f $PIDFILE ]
           then
                   echo "$PIDFILE exists, process is already running or crashed"
           else
                   echo "Starting Redis server..."
                   $EXEC $CONF
           fi
           ;;
       stop)
           if [ ! -f $PIDFILE ]
           then
                   echo "$PIDFILE does not exist, process is not running"
           else
                   PID=$(cat $PIDFILE)
                   echo "Stopping ..."
                   $CLIEXEC -p $REDISPORT shutdown
                   while [ -x /proc/${PID} ]
                   do
                       echo "Waiting for Redis to shutdown ..."
                       sleep 1
                   done
                   echo "Redis stopped"
           fi
           ;;
       *)
           echo "Please use start or stop as first argument"
           ;;
   esac
   ```

   根据启动脚本配置文件复制到指定目录之下

# redis数据类型

## 字符串（String）

字符串是redis最基本的数据类型，一个键最多存储512MB。

```
127.0.0.1:6379> set bsxrb helloredis
OK
127.0.0.1:6379> get bsxrb
"helloredis"
127.0.0.1:6379> 

```

## 列表（Lists）

redis列表元素 为字符串，按照插入顺序排序，底层通过双向列表来实现。你可以通过左插，右插来实现双向队列。常用操作包括` LPUSH`(左插) ，` RPUSH`（右插），` LRANGE` （获取左边多少个元素）等。

```
127.0.0.1:6379> lpush blist a
(integer) 1
127.0.0.1:6379> lpush blist b
(integer) 2
127.0.0.1:6379> rpush blist c
(integer) 3
127.0.0.1:6379> rpush blist d
(integer) 4
127.0.0.1:6379> lrange blist 0 3
1) "b"
2) "a"
3) "c"
4) "d"
127.0.0.1:6379> lpop blist
"b"
127.0.0.1:6379> rpop blist
"d"

```

[更多lists命令](https://www.runoob.com/redis/redis-lists.html)

列表应用

* 实现简单的消息队列功能
* 通过lrange实现排行榜以及分页功能

## 集合（Sets）

set是无序集合最多可以 添加2^32-1个元素，set通过 hash table实现，hash table的大小 会随着元素的增加和删除调整。由于redis是单线程的所以在调整hash table大小是可能会导致导致堵塞之后的操作。常用操作包括` SADD` 集合添加`	SREM`移除集合成员 `	SUNION` 返回并集`	SINTER` 返回交集等

```
127.0.0.1:6379> sadd fset a
(integer) 1
127.0.0.1:6379> sadd fset b c
(integer) 2
127.0.0.1:6379> smembers fset
1) "b"
2) "c"
3) "a"
127.0.0.1:6379> sadd sset a e f
(integer) 3
127.0.0.1:6379> sunion sset fset
1) "a"
2) "e"
3) "b"
4) "c"
5) "f"
127.0.0.1:6379> sinter sset fset
1) "a"
```

[更多Sets命令](https://www.runoob.com/redis/redis-sets.html)

Sets 的应用场景

- 好友/关注/粉丝/感兴趣的人集合

- 随机展示 

- 黑名单/白名单

## 有序集合(sorted sets)

有序集合指的是每个元素都有个序号（double且可重复），根据序号进行排序。redis中的有序集合也被称为zsets。

zsets的底层实现可能有多种，redis目前使用的编码方式包括：

```c
/* Objects encoding. Some kind of objects like Strings and Hashes can be
 * internally represented in multiple ways. The 'encoding' field of the object
 * is set to one of this fields for this object. 
 */
#define OBJ_ENCODING_RAW     /* Raw representation */ 简单动态字符串
#define OBJ_ENCODING_INT      /* Encoded as integer */ 整数
#define OBJ_ENCODING_HT       /* Encoded as hash table */ 字典
#define OBJ_ENCODING_ZIPLIST  /* Encoded as ziplist */ 压缩列表
#define OBJ_ENCODING_INTSET   /* Encoded as intset */ 整数集合
#define OBJ_ENCODING_SKIPLIST   /* Encoded as skiplist */ 跳跃表
#define OBJ_ENCODING_EMBSTR  /* Embedded sds string encoding */ embstr编码的简单动态字符串
#define OBJ_ENCODING_QUICKLIST  /* Encoded as linked list of ziplists */
```

zset对象的编码可以是ziplist 或者skiplist 。当元素数量小于128个，切所有成员长度都小于64字节时使用ziplist(可以通过zmax-ziplist-entries  和 zset-max-ziplist-value来改)[ziplist详情](https://blog.csdn.net/yellowriver007/article/details/79021049)

除此之外使用skiplist结合dict[skiplist详情](http://zhangtielei.com/posts/blog-redis-skiplist.html)和ziplist

常用操作包括`	zadd`(添加元素)`	zcount`（计算有序集合指定区间成员数）`	zrange`(通过索引区间返回有序集合指定区间内的成员)等。 

```
127.0.0.1:6379> zadd bzset 1 a
(integer) 1
127.0.0.1:6379> zadd bzset 2  b  3  c 4  d
(integer) 3
127.0.0.1:6379> zadd bzset e e
(error) ERR value is not a valid float
127.0.0.1:6379> zrange bzset 0 4 withscores
1) "a"
2) "1"
3) "b"
4) "2"
5) "c"
6) "3"
7) "d"
8) "4"
127.0.0.1:6379> 
```

[更多zset命令 ](https://www.runoob.com/redis/redis-sorted-sets.html)

Zset使用场景

有排序的各种列表如

* 积分排行榜
* 时间排行列表等

## 哈希（Hashes）

哈希存储的是键值对，比较适合存储对象。redis中每个hash可以存2^32-1个hash

常用操作又`	HMSET`插入一个hash ` HGETALL`获取一个hash`	HMGET`获取一个hash中的某个字段。

```
127.0.0.1:6379> HMSET st1 name bsx age 20 sex male
OK
127.0.0.1:6379> hmset ste name bsr age 12 sex male
OK
127.0.0.1:6379> hmset st3 name bsb age 21 sex female
OK
127.0.0.1:6379> hmset st4 name bsc age 19 sex female
OK
127.0.0.1:6379> hgetall st1
1) "name"
2) "bsx"
3) "age"
4) "20"
5) "sex"
6) "male"
127.0.0.1:6379> hget  st4 name
"bsc"
127.0.0.1:6379> 

```

hash使用场景

* 购物车
* 存储对象(也可以用String+json看使用场景)

## Hyperloglog

Redis HyperLogLog 是用来做基数统计的算法，HyperLogLog 的优点是，在输入元素的数量或者体积非常非常大时，计算基数所需的空间总是固定 的、并且是很小的。

在 Redis 里面，每个 HyperLogLog 键只需要花费 12 KB 内存，就可以计算接近 2^64 个不同元素的基 数。这和计算基数时，元素越多耗费内存就越多的集合形成鲜明对比。

但是，因为 HyperLogLog 只会根据输入元素来计算基数，而不会储存输入元素本身，所以 HyperLogLog 不能像集合那样，返回输入的各个元素

常用操作`	pfadd`添加元素到Hyperloglog中`	pfcount`得到基数估算值。

```
127.0.0.1:6379> pfadd  bsxrb a
(integer) 1
127.0.0.1:6379> pfadd bsxrb b
(integer) 1
127.0.0.1:6379> pfadd bsxrb c
(integer) 1
127.0.0.1:6379> pfadd bsxrb d
(integer) 1
127.0.0.1:6379> pfadd bsxrb a
(integer) 0
127.0.0.1:6379> pfcount bsxrb
(integer) 4
127.0.0.1:6379> 

```

hyperloglog 是一种估算基数的算法。具体等有时间再研究研究。

使用场景

* 统计ip数
* 计算UV数等

# Redis  数据持久化

redis 为了性能将数据都存放在内存之中，所以需要持久化机制来保存数据。

redis持久化主要有两种

## RDB

RDB持久化可以通过save 和 BGsave命令来手动持久化，也可以 根据配置信息定期执行 。定期执行也是本质上也是执行BSSAVE命令。

SAVA命令会阻塞redis服务器进程，BGSAVE则是通过创建一个子进程，通过子进程创建RDB文件。

还可以通过save设置保存条件，满足其中任意一个就会执行BGSAVE命令,如：

save 60 1 #在60秒内对数据库至少进行1次修改，原理是通过周期性操作函数serverCorn,会在默认100ms检查一次看是否满足设置的save条件。

redis服务器维持了一个dirty计数器以及lastsave属性其中

dirty计数器记录了距离上一次save 或者bgsave 执行你了多少次修改(增 删 改)

lastsave是一个UNIX时间戳，记录上一次修改的时间。 

## AOF

AOF类似mysql中的binlog原理是把所有修改操作都记录下来，重启服务是根据aof文件来还原数据。

例如：

```
127.0.0.1:6379> set bsxrb aaa
OK
127.0.0.1:6379> hset bsxrb name llc
(error) WRONGTYPE Operation against a key holding the wrong kind of value
127.0.0.1:6379> hset hbsxrb name llc
(integer) 1
127.0.0.1:6379> lpush blist atest
(integer) 1

```

当我们通过修改配置文件appdonly yes开启aof持久化之后，执行上诉插入操作，会生成一个aof文件

```
*2
$6
SELECT
$1
0
*3
$3
set
$5
bsxrb
$3
aaa
*4
$4
hset
$6
hbsxrb
$4
name
$3
llc
*3
$5

```

记录下了我们的所有操作。

默认下AOF持久化策略是每秒同步一次。

如果在追加日志时，恰好遇到磁盘空间满、inode满或断电等情况导致日志写入不完整，也没有关系，redis提供了redis-check-aof工具，可以用来进行日志修复。

因为采用了追加方式，如果不做任何处理的话，AOF文件会变得越来越大，为此，redis提供了AOF文件重写（rewrite）机制，即当AOF文件的大小超过所设定的阈值时，redis就会启动AOF文件的内容压缩，只保留可以恢复数据的最小指令集。举个例子或许更形象，假如我们调用了100次INCR指令，在AOF文件中就要存储100条指令，但这明显是很低效的，完全可以把这100条指令合并成一条SET指令，这就是重写机制的原理。

Redis在对AOF文件重写时， 并不需要对现有的AOF文件进行任何读取、分析或者写入操作，而是通过读取服务器当前的数据库状态来实现的。比如前面说的，调用了100次INCR指令，Redis并不会遍历分析这100条指令，而是简单地读取了当前数据库中该键对应的值，然后使用一条set指令将这个值写入到新的AOF文件中。这就是AOF重写功能的实现原理。

AOF方式的另一个好处，我们通过一个“场景再现”来说明。某同学在操作redis时，不小心执行了FLUSHALL，导致redis内存中的数据全部被清空了，这是很悲剧的事情。不过这也不是世界末日，只要redis配置了AOF持久化方式，且AOF文件还没有被重写（rewrite），我们就可以用最快的速度暂停redis并编辑AOF文件，将最后一行的FLUSHALL命令删除，然后重启redis，就可以恢复redis的所有数据到FLUSHALL之前的状态了。是不是很神奇，这就是AOF持久化方式的好处之一。但是如果AOF文件已经被重写了，那就无法通过这种方法来恢复数据了。

虽然优点多多，但AOF方式也同样存在缺陷，比如在同样数据规模的情况下，AOF文件要比RDB文件的体积大。而且，AOF方式的恢复速度也要慢于RDB方式。

AOF也是通过Fork一个子进程实现的。

