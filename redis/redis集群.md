# redis集群

## 主从模式

主从模式是三种模式中最简单的，在主从复制中，数据库分为两类：主数据库(master)和从数据库(slave)。而且支持一主多从。主从结构一是为了纯粹的冗余备份，二是为了提升读性能，三是为了增加可靠性 。主从架构中，可以考虑关闭主服务器的数据持久化功能，只让从服务器进行持久化从而提升主服务器的性能。

### 搭建一主两从

在三台虚拟机上都安装好redis并配置好网络环境：

master `	192.168.52.130`

slaver01`	192.168.52.101`

slaver02`	192.168.52.102`

修改redis.conf

关闭master的数据持久化，开启两台从机的数据持久化，并将三台机器同时开启deamonize模式 

从机将replaceof设置为 主机的 ip和端口号，并输入主机密码。同时开启三台 redis服务，简单的主从 模式就配置好了。

```bash
replicaof 192.168.52.130 6379

# If the master is password protected (using the "requirepass" configuration
# directive below) it is possible to tell the replica to authenticate before
# starting the replication synchronization process, otherwise the master will
# refuse the replica request.
#
masterauth llc782923640+

```

测试一下主机中set一个String

```
127.0.0.1:6379> auth llc782923640+
OK
127.0.0.1:6379> set bsxrb a
OK

```

两台从机都能成功得到配置成功。

```
[root@slaver01 redis-5.0.3]# src/redis-cli
127.0.0.1:6379> get bsxrb
"a"
[root@slaver02 redis-5.0.3]# src/redis-cli
127.0.0.1:6379> get bsxrb
"a"

```

redis的复制功能分为初次同步和命令传播两个阶段。

#### 初次同步

当主从第一次连接时从服务器会向主服务器发送psync命令，主服务收到之后执行bgsave，生成一个RDB文件，并使用新建一个缓冲区保存之后的写命令。从服务使用主服务发送来的RDB文件及之后缓冲区的写命令来恢复成主服务器的状态。又有一个自己的ID由40个随机十六进制字符组成

#### 命令传播

完成初步同步之后，主服务器会将 所有的写命令发送给从服务器，从而实现主从同步。

#### 部分重同步



主从服务器断开重连后，在2.8版本之前会进行全量同步。这无疑是非常消耗资源的，2.8之后redis更新了增量同步策略。所有redis服务有一个自己的ID由40个随机十六进制字符组成，当第一次主从同步完成之后从服务器会记录主服务器的ID，当断开重连之后如果从服务器存的id和主服务器相同就会进行部分同步。部分同步机制除了id之外依赖两个关键部分：

##### 偏移量

主服务器 每次向从服务器传播数据时都会记录偏移量，传播了N个字节 就在偏移量上加N从服务其接受主服务器传播的字节时也记录偏移量，当两个偏移量相等时表示状态一致

##### 积压缓冲区

挤压缓冲区是有主服务器维护的一个队列，默认大小1M当主服务器向从服务器发送命令时，还会将命令写入积压缓冲区中，积压缓冲区 记录着每个字节的值和他的偏移量。当从服务器重连主服务器之后，从服务器通过psync命令将自己的偏移量发送个主服务器，如果从服务器发送的偏移量能在积压缓冲区中找到时，就从该字符开始将之后的命令发送给从服务器，以此来达到主从一致。否则重新进行全量完整 同步操作。

## 哨兵模式



Sentinel(哨兵)是用于监控redis集群中Master状态的工具，是Redis 的高可用性解决方案，sentinel哨兵模式已经被集成在redis2.4之后的版本中。sentinel是redis高可用的解决方案，sentinel系统可以监视一个或者多个redis master服务，以及这些master服务的所有从服务；当某个master服务下线时，自动将该master下的某个从服务升级为master服务替代已下线的master服务继续处理请求。

sentinel可以让redis实现主从复制，当一个集群中的master失效之后，sentinel可以选举出一个新的master用于自动接替master的工作，集群中的其他redis服务器自动指向新的master同步数据。一般建议sentinel采取奇数台，防止某一台sentinel无法连接到master导致误切换。

### Sentinel工作流程

1. 每个Sentinel以每秒钟一次的频率向它所知的Master，Slave以及其他 Sentinel 实例发送一个PING命令。
2. 如果一个实例（instance）距离最后一次有效回复PING命令的时间超过 own-after-milliseconds 选项所指定的值，则这个实例会被Sentinel标记为主观下线。 
3. 如果一个Master被标记为主观下线，则正在监视这个Master的所有 Sentinel 要以每秒一次的频率确认Master的确进入了主观下线状态。 
4. 当有足够数量的Sentinel（大于等于配置文件指定的值）在指定的时间范围内确认Master的确进入了主观下线状态，则Master会被标记为客观下线。

### 主观下线

主观下线指的是一个Sentinel对某个服务器发送ping命令后，在指定的时间内(down-after-milliseconds)没有收到pong命令，或者返回错误命令则该Sentinel标记该服务为主观下线(SRI_S_DOWN)

### 客观下线

客观下线(SDOWN)是指多个sentinel实例对master服务器做出了主观下线判断后，该服务器会被判定为客观下线。

当sentinel监视某个服务主观下线之后，sentinel会询问其他监视该服务的sentinel，看他们是否认为服务主观下线，在收到足够数量之后(一般配置为n/2+1)即可对该服务判定为客观下线，并对其进行故障转移操作。



### 配置传播

一旦一个sentinel成功地对一个master进行了failover，它将会把关于master的最新配置通过广播形式通知其它sentinel，其它的sentinel则更新对应master的配置。
一个faiover要想被成功实行，sentinel必须能够向选为master的slave发送SLAVEOF NO ONE命令，然后能够通过INFO命令看到新master的配置信息。
当将一个slave选举为master并发送SLAVEOF NO ONE后，即使其它的slave还没针对新master重新配置自己，failover也被认为是成功了的，然后所有sentinels将会发布新的配置信息。
新配在集群中相互传播的方式，就是为什么我们需要当一个sentinel进行failover时必须被授权一个版本号的原因。
每个sentinel使用##发布/订阅##的方式持续地传播master的配置版本信息，配置传播的##发布/订阅##管道是：__sentinel__:hello。
因为每一个配置都有一个版本号，所以以版本号最大的那个为标准。

### sentinel选举

一个redis服务被判断为客观下线时，多个监视该服务的sentinel协商，选举一个领头sentinel，对该redis服务进行故障转移操作。**选举领头sentinel遵循以下规则：**
1）所有的sentinel都有公平被选举成领头的资格。
2）所有的sentinel都有且只有一次将某个sentinel选举成领头的机会（在一轮选举中），一旦选举某个sentinel为领头，不能更改。
3）sentinel设置领头sentinel是先到先得，一旦当前sentinel设置了领头sentinel，以后要求设置sentinel为领头请求都会被拒绝。
4）每个发现服务客观下线的sentinel，都会要求其他sentinel将自己设置成领头。
5）当一个sentinel（源sentinel）向另一个sentinel（目sentinel）发送is-master-down-by-addr ip port current_epoch runid命令的时候，runid参数不是*，而是sentinel运行id，就表示源sentinel要求目标sentinel选举其为领头。
6）源sentinel会检查目标sentinel对其要求设置成领头的回复，如果回复的leader_runid和leader_epoch为源sentinel，表示目标sentinel同意将源sentinel设置成领头。
7）如果某个sentinel被半数以上的sentinel设置成领头，那么该sentinel既为领头。
8）如果在限定时间内，没有选举出领头sentinel，暂定一段时间，再选举。

**为什么要选领导者？**
简单来说，就是因为只能有一个sentinel节点去完成故障转移。
sentinel is-master-down-by-addr这个命令有两个作用，一是确认下线判定，二是进行领导者选举。
**选举过程：**

1. 每个做主观下线的sentinel节点向其他sentinel节点发送上面那条命令，要求将它设置为领导者。
2. 收到命令的sentinel节点如果还没有同意过其他的sentinel发送的命令（还未投过票），那么就会同意，否则拒绝。
3. 如果该sentinel节点发现自己的票数已经过半且达到了quorum的值，就会成为领导者
4. 如果这个过程出现多个sentinel成为领导者，则会等待一段时间重新选举。 

### 主从切换方案

