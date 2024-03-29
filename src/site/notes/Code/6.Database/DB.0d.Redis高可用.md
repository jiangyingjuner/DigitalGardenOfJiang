---
{"dg-publish":true,"permalink":"/Code/6.Database/DB.0d.Redis高可用/","title":"Redis 高可用","noteIcon":""}
---


# Redis 高可用

## 主从复制

> [主从复制是怎么实现的？ | 小林coding (xiaolincoding.com)](https://xiaolincoding.com/redis/cluster/master_slave_replication.html)

通过将数据备份到其他服务器上可避免**单点故障问题**

Redis 提供**主从复制模式**以解决多服务器间的数据一致性问题

主服务器可以进行读写操作，当发生写操作时自动将写操作同步给从服务器
从服务器一般只读，同时接受主服务器同步过来的写操作命令并执行

Redis 使用 `replicaof` 命令形成主服务器和从服务器的关系
```
# 在slave服务器执行命令
replicaof {master 的 IP 地址} {slave 的 Redis 端口号}
```

主从复制共有三种形式：**全量复制、基于长连接的命令传播、增量复制**

### 全量复制

主从服务器间的*第一次同步的过程*可分为三个阶段：
1. 建立链接、协商同步
    1. 从服务器向主服务器发送 `psync` 命令
    2. 主服务器返回 `FULLRESYNC` 响应命令，表示采用**全量复制方式**同步数据
2. 主服务器同步数据给从服务器
    1. 主服务器执行 BGSAVE 命令生成 RDB 文件并发送给从服务器
    2. 从服务器收到后清空当前数据，载入 RDB 文件，同时主服务器将以下三个时期中收到的写操作命令，写入到 *replication buffer* 缓冲区里
        - 主服务器生成 RDB 期间
        - 主服务器发送 RDB 期间
        - 从服务器加载 RDB 期间
    3. 从服务器完成载入 RDB 后回复确认消息给主服务器
3. 主服务器发送新写操作命令给从服务器
    1. 主服务器将 `replication buffer` 缓冲区内记录的写操作命令发送给从服务器
    2. 从服务器依次执行命令，完成第一次同步

从服务器发送给主服务器的 `psync` 命令中包含*主服务器的 runID* 和*复制进度 offset*
- `runID`，每个 Redis 服务器启动时自动生产一个随机的唯一自我标识 ID ，第一次同步时从服务器不知道主服务器的 run ID，所以将其设置为 `?`
- `offset`，表示复制的进度，第一次同步时，其值为 -1

> ![ea4f7e86baf2435af3999e5cd38b6a26.png (1080×555) (xiaolincoding.com)|675](https://cdn.xiaolincoding.com//mysql/other/ea4f7e86baf2435af3999e5cd38b6a26.png)

### 基于长连接的命令传播

主从服务器在完成第一次同步后维护一个 *TCP 长连接*
主服务器通过该连接继续将写操作命令传播给从服务器，然后从服务器执行该命令，保持与主服务器的数据库状态相同

通过*基于长连接的命令传播*保证了第一次同步后的主从服务器的数据一致性

*过期 key*：主节点处理了一个 key 或者通过淘汰算法淘汰了一个 key，这个时间主节点模拟一条 del 命令发送给从节点，从节点收到该命令后，就进行删除 key 的操作

### 增量复制

主从服务器重连时可以采用*增量复制*的方式继续同步，仅将网络断开期间主服务器接收到的写操作命令同步给从服务器

增量复制步骤：
1. 从服务器在恢复网络后发送 `psync` 命令给主服务器
2. 主服务器用 `CONTINUE` 响应命令，表示采用**增量复制的方式**同步数据
3. 主服务器发送网络断开期间主服务器接收的写操作命令，从服务器执行命令

具体的增量数据通过 *repl_backlog_buffer* 和 *replication offset* 确认

repl_backlog_buffer 是一个 **环形缓冲区**，Redis 主节点每次收到写命令之后，先写到内部的 repl_backlog_buffer 缓冲区，然后**异步发送给从节点**
主节点使用 `master_repl_offset` 标记缓冲区中当前写入的位置，从节点使用 `slave_repl_offset` 标记缓冲区中当前读出的位置

增量复制过程中从服务器通过 `psync` 命令发送自己的复制偏移量 `slave_repl_offset`，主服务器根据 `master_repl_offset` 和 `slave_repl_offset` 之间的差距决定执行哪种同步操作
- `slave_repl_offset` 还在 `repl_backlog_buffer` 缓冲区里，采用增量同步
- `slave_repl_offset` 已不在缓冲区里，采用全量同步

> ![2db4831516b9a8b79f833cf0593c1f12.png (1080×380) (xiaolincoding.com)](https://cdn.xiaolincoding.com//mysql/other/2db4831516b9a8b79f833cf0593c1f12.png)

为了避免在网络恢复时主服务器频繁进行全量同步，应该使 `repl_backlog_buffer` 缓冲区大小尽可能大

#### Replication buffer VS repl backlog buffer

- 出现阶段与数量不同：
    - repl backlog buffer 是在增量复制阶段出现，一个主节点只分配一个 repl backlog buffer
    - replication buffer 是在全量复制阶段和增量复制阶段都会出现，主节点会给每个新连接的从节点，分配一个 replication buffer
- 溢出处理不同：
    - repl backlog buffer 环形缓冲区溢出的新数据会直接覆盖起始位置数据
    - replication buffer 溢出导致连接断开，删除缓存，从节点重新连接，重新进行全量复制

### 异常

#### 主从断连

Redis 通过互相的 ping-pong 心跳检测机制检测断连，如果一半以上的节点去 ping 一个节点的时候没有收到 pong 回应，集群认为这个节点挂掉并断开与这个节点的连接

- Redis 主节点默认每隔 10 秒对从节点发送 ping 命令判断存活性和连接状态，可通过参数 `repl-ping-slave-period` 控制发送频率
- Redis 从节点每隔 1 秒向主节点发送 `replconf ack{offset}` 命令上报自身当前的复制偏移量
    - 实时监测主从节点网络状态
    - 检查复制数据是否丢失，如果从节点数据丢失，再从主节点的replication buffer缓冲区中拉取丢失数据

#### 主从数据不一致

主从节点之间的命令复制异步进行，因此无法保证主从数据的强一致性

解决方法：
- 保证主从服务器连接状态良好
- 使用外部程序检查主从服务器 `repl offset` 差值，对客户端屏蔽差值较大的从服务器

#### 主从数据丢失

主从切换过程中，产生数据丢失的情况有两种：
- 异步复制同步丢失
- 集群产生脑裂数据丢失

##### 异步复制命令丢失

问题：
主服务器已进行写操作命令但还未将命令异步发送给从服务器时宕机

解决方案：
通过参数 `min-slaves-max-lag` 定义主从服务器间的最大差异时间，在主从节点间数据差异过大(所有的从节点数据复制和同步的延迟都超过了设定值)时主服务器拒绝写入新请求
客户端可将数据暂时写入本地缓存或消息队列中

##### 集群脑裂导致数据丢失

> [xiaolincoding.com](https://xiaolincoding.com/redis/cluster/master_slave_replication.html#%E9%9B%86%E7%BE%A4%E4%BA%A7%E7%94%9F%E8%84%91%E8%A3%82%E6%95%B0%E6%8D%AE%E4%B8%A2%E5%A4%B1)

问题：
主节点与从节点失联但与客户端连接正常，重新上线时变为新主节点的从节点，清空数据重新全量同步，导致失联后客户端写入的数据丢失

解决方案：
当主节点发现**从节点下线的数量太多**或**网络延迟太大**时，主节点会禁止写操作，直接把错误返回给客户端

### 主服务器减压

问题：
- 从服务器数量多时，主服务器忙于使用 fork()创建子进程，进而导致主线程阻塞
- 传输 RDB 文件占用主服务器网络带宽，影响主服务器响应命令
解决方法：
- 令从服务器创建从服务器，分担主服务器压力

> ![4d850bfe8d712d3d67ff13e59b919452.png (1052×632) (xiaolincoding.com)|600](https://cdn.xiaolincoding.com//mysql/other/4d850bfe8d712d3d67ff13e59b919452.png)

### 主从复制缺点

- Redis 不具备自动容错和恢复功能，主机从机的宕机都会导致前端部分读写请求失败，需要等待机器重启或者手动切换前端的 IP 才能恢复(**也就是要人工介入**)；
- 主机宕机，宕机前有部分数据未能及时同步到从机，切换IP后还会引入数据不一致的问题，降低了系统的可用性；
- 如果多个 Slave 断线了，需要重启的时候，尽量不要在同一时间段进行重启。因为只要 Slave 启动，就会发送sync 请求和主机全量同步，当多个 Slave 重启的时候，可能会导致 Master IO 剧增从而宕机。
- Redis 较难支持在线扩容，在集群容量达到上限时在线扩容会变得很复杂；

## 哨兵

哨兵的作用是实现**主从节点故障转移**，哨兵监测主节点是否存活，如果发现主节点挂了，哨兵选举一个从节点切换为主节点，并且把新主节点的相关信息通知给从节点和客户端

> ![|675](https://cdn.xiaolincoding.com/gh/xiaolincoder/redis/%E5%93%A8%E5%85%B5/%E4%B8%BB%E4%BB%8E%E6%95%85%E9%9A%9C%E8%BD%AC%E7%A7%BB.png)

哨兵主要负责*监控*、*选主*、*通知*

### 监控

哨兵每隔 1 秒给所有主从节点以及其他哨兵节点发送 PING 命令，当节点收到 PING 命令后发送一个响应命令给哨兵，这样就可以判断它们是否在正常运行
哨兵将没有在规定的时间内响应 PING 命令的节点标记为*主观下线*，规定时间通过配置项 `down-after-milliseconds` 参数设定

哨兵用多个节点部署成*哨兵集群*(最少三台机器)以减少误判，避免因为单个哨兵自身网络状况不好而误判主节点下线的情况
*客观下线*仅针对主节点
1. 当一个哨兵判断主节点*主观下线*后，向其他哨兵发起询问命令
2. 其他哨兵根据自身连接情况对该判断投票
3. 赞同票数达到设定值 `quorum` 时，主节点被1中的哨兵标记为*客观下线*
    - `quorum` 的值一般设置为哨兵个数的1/2加1

### 选主

#### 哨兵leader

在哨兵集群中，由 leader 哨兵进行主从故障转移
Leader 通过投票选举：
1. 在主节点客观下线后，标记主节点的哨兵成为 leader 候选者
2. 候选者会向其他哨兵发送命令，表明希望成为 leader 来执行主从切换，并让所有其他哨兵对它进行投票
    - 每个哨兵只有一次投票机会，投给先收到其投票请求的候选者，候选者给自己投票
3. 满足条件的候选者成为 leader
    - 拿到半数以上的赞成票
    - 拿到的票数大于等于哨兵配置文件中的 `quorum` 值

哨兵节点应设置为奇数个，同时 `quorum` 的值设置为哨兵个数的1/2加1以保证主节点客观下线后能够选出哨兵 leader

#### 选出新主节点

> ![|500](https://cdn.Xiaolincoding.Com/gh/xiaolincoder/redis/%E5%93%A8%E5%85%B5/%E9%80%89%E4%B8%BB%E8%BF%87%E7%A8%8B.webp)

*优先级*：Redis 通过配置项` slave-priority` 给从节点设置优先级

选举出从节点后，哨兵 leader 向被选中的从节点发送 `SLAVEOF no one` 命令，让这个从节点解除从节点的身份，将其变为新主节点
之后哨兵 leader 以每秒一次的频率向被升级的从节点发送 `INFO` 命令，并观察命令回复中的角色信息，当被升级节点的*角色信息*从原来的 slave 变为 master 时，哨兵 leader 就知道被选中的从节点已经顺利升级为主节点了

正常状态下，哨兵以每 10s 一次的频率向主节点发送 `INFO` 命令来获取所有从节点的信息

### 通知

被选中的从节点顺利升级后，哨兵 leader 向所有从节点发送 `SLAVEOF` 命令，让它们成为新主节点的从节点

哨兵集群通过*发布者/订阅者机制*向客户端提供新主节点的信息，主从切换完成后，哨兵就会向 `+switch-master` 频道发布新主节点的 IP 地址和端口的消息，这个时候客户端就可以收到这条信息，然后用这里面的新主节点的 IP 地址和端口进行通信了

最后哨兵集群监视旧主节点，当其重新上线时向其发送 `SLAVEOF` 命令，让其成为新主节点的从节点，结束主从节点的故障转移的工作

哨兵提供的消息订阅频道有很多，不同频道包含了主从节点切换过程中的不同关键事件
> ![|600](https://cdn.xiaolincoding.com/gh/xiaolincoder/redis/%E5%93%A8%E5%85%B5/%E5%93%A8%E5%85%B5%E9%A2%91%E9%81%93.webp)

### 哨兵集群

哨兵配置
```
sentinel monitor {master-name} {ip} {redis-port} {quorum} 
```

哨兵节点之间**通过 Redis 的发布者/订阅者机制相互发现**

在主从集群中，主节点上有一个名为 `__sentinel__:hello` 的频道，哨兵节点将自己的 IP 地址和端口信息发布到该频道上后，其他哨兵就可以获取这些信息，从而建立网络连接

## Cluster 集群

> [[Redis] 你了解 Redis 的三种集群模式吗？ - 个人文章 - SegmentFault 思否](https://segmentfault.com/a/1190000022808576)

Redis 的哨兵模式基本已经可以实现高可用，读写分离，但是在这种模式下每台 Redis 服务器都存储相同的数据，很浪费内存
Redis3.0上加入了 Cluster 集群模式，实现了 Redis 的分布式存储，**每台 Redis 节点上存储不同的内容**，以此来降低系统对单主节点的依赖，并且可以大大的提高 Redis 服务的读写性能

> ![image-20200531184321294](https://segmentfault.com/img/remote/1460000022808584 "image-20200531184321294")

图中每一个蓝色的圈都代表着一个 redis 的服务器节点，任何两个节点之间相互连通的，客户端与任何一个节点相连接后就可以访问集群中的任何一个节点，对其进行存取和其他操作

### 哈希槽

Redis 集群没有使用一致性 hash，而是引入了**哈希槽*hash slot***的概念

```C
// from redis 5.0.5
#define CLUSTER SLOTS 16384
typedef struct clusterNode {
  ...
  unsigned char slots[CLUSTER_SLOTS/8]; /* slots handled by this node */
  int numslots; /* Number of slots handled by this node */
  ...
} clusterNode;
```

Redis 集群共有16384 个哈希槽，每个 key 通过 CRC16 校验后对 16384 取模来决定放置哪个槽

集群的每个节点都包含一个表示整个槽空间的 `slots` ，类似于位图，用对应槽位的 1/0 区分是否为自己负责的 hash 槽
这种位图的形式可以使节点在 O(1) 时间复杂度下判断指定槽是否由自己负责

这种结构很容易添加或者删除节点。比如如果我想新添加个节点 D ，我需要从节点 A， B， C 中得部分槽到 D 上。如果我想移除节点 A ，需要将 A 中的槽移到 B 和 C 节点上，然后将没有任何槽的 A 节点从集群中移除即可。由于从一个节点将哈希槽移动到另一个节点并不会停止服务，所以无论添加删除或者改变某个节点的哈希槽的数量都不会造成集群不可用的状态。

在分配槽之后，节点通过向集群中其他节点放送消息告知负责了哪些槽，每个节点中维护了一个 clusterState 结构，记录了每个槽由哪个节点负责
当客户端请求错误节点时，节点返回应访问的正确节点

### 主从复制

为了保证高可用，redis-cluster 集群引入了主从复制模型，一个主节点对应一个或者多个从节点，当主节点宕机的时候，就会启用从节点。当其它主节点 ping 一个主节点 A 时，如果半数以上的主节点与 A 通信超时，那么认为主节点 A 宕机了。如果主节点 A 和它的从节点 A1 都宕机了，那么该集群就无法再提供服务了。

### 特点

- 所有的 redis 节点彼此互联(PING-PONG机制)，内部使用二进制协议优化传输速度和带宽。
- 节点的 fail 是通过集群中超过半数的节点检测失效时才生效
- 客户端与 Redis 节点直连，不需要中间代理层，客户端不需要连接集群所有节点，连接集群中任何一个可用节点即可
