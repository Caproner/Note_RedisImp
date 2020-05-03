# 第17章 集群

Redis支持集群模式。不同于主从服务器模式，集群模式将数据分为许多片，每个服务器（在集群中称之为节点）管理一到多个片。这样的分布式架构可以做到分摊写命令的流量。

## 节点

在集群模式中，每个服务器被视为一个节点。一开始每个节点之间都是独立的。客户端通过发送如下命令让节点和目标节点连接起来：

```
CLUSTER MEET <ip> <port>
```

通过n-1次发送该命令，可以将n个节点连接起来建立一个集群。

接下来讲节点及节点连接的原理

### 启动节点

当需要使用集群的时候，需要将服务器以集群模式启动。这由`cluster-enabled`配置决定。

服务器在集群模式下作为一个节点被启动。其初始化所做的事情跟启动一个普通的服务器基本一致。除此之外其多初始化了三种集群数据结构用于存储集群模式下才会用到的信息。

### 集群数据结构

`clusterNode`用于记录节点的基础信息：

```c
struct clusterNode {
  // 创建时间
  mstime_t ctime;
  // 节点名字，为一个40位十六进制数字
  char name[REDIS_CLUSTER_NAMELEN];
  // 节点标记，用于标记其各种状态
  int flags;
  // 节点当前配置纪元，故障转移用
  uint64_t configEpoch;
  // 节点IP
  char ip[REDIS_IP_STR_LEN];
  // 节点端口号
  int port;
  // 保存节点连接相关信息
  clusterLink *link;
  // ...
};

typedef struct clusterLink {
  // 创建时间
  mstime_t ctime;
  // 套接字
  int fd;
  // 输出缓冲区
  sds sndbuf;
  // 输入缓冲区
  sds rcvbuf;
  // 与这个链接相关的节点
  struct clusterNode *node;
} clusterLink;
```

节点使用一个`clusterState`结构体实例保存自身的节点信息和与其相连的节点的信息：

```c
typedef struct clusterState {
  // 指向当前节点的指针
  clusterNode *myself;
  // 集群当前的配置纪元，故障转移用
  uint64_t currentEpoch;
  // 集群当前状态（在线/下线）
  int state;
  // 集群的有效节点数量（有效节点=在处理至少一个槽的节点）
  int size;
  // 集群的节点字典，其中包括自身
  // 键为节点名，值为clusterNode指针
  dict *nodes;
  // ...
} clusterState;
```

### CLUSTER MEET的实现

假设客户端向节点A发送`CLUSTER MEET`命令，参数指向节点B。

节点A收到命令之后会跟节点B进行握手。过程如下：

+ 节点A在`nodes`字典里为节点B创建一个`clusterNode`结构
+ 节点A给节点B发送一条`MEET`消息（集群消息在之后会讲到）
+ 节点B接收到消息，在字典里为节点A创建一个`clusterNode`结构
+ 节点B给节点A发送`PONG`消息
+ 节点A给节点B发送`PING`消息
+ 节点B收到消息，握手完成

握手完成之后，节点A会将节点B的信息传播给集群中的其他节点，让其他节点分别跟节点B握手，最终使节点B被节点A所在的集群的所有节点认识。

> 这么一来有个问题。假设节点B本身就属于一个集群，这个时候节点A拉节点B入伙的话，有两种理解：
>
> + 将节点B从原集群中拉出来，那么怎么将原集群的节点信息从节点B的字典中清除，且原集群的其他节点怎么知道节点B被拉出来呢？
> + 将两个集群合并，那么节点A所在的集群要怎么跟节点B所在的集群相互认识呢？

## 槽指派

在Redis集群模式中，分片策略为：

+ 将键值进行哈希运算得到一个0~16383的值，然后将这个键值对放入对应编号的槽中
+ 一个槽只属于一个节点，每个节点可以有多个槽

这种分片策略会涉及到将槽分给谁的问题，也就是槽指派。

我们可以向节点发送`CLUSTER ADDSLOT`命令给节点分配槽：

```
127.0.0.1:7000> CLUSTER ADDSLOT 0 1 2 3 4 ... 5000
...
127.0.0.1:7001> CLUSTER ADDSLOT 5001 5002 5003 5004 ... 10000
...
127.0.0.1:7002> CLUSTER ADDSLOT 10001 10002 10003 1004 ... 16383
...
```

当且仅当每个槽都有节点在管理的时候，集群为上线状态，否则其为下线状态。

### 记录节点的槽指派信息

每个`clusterNode`结构体中会有一个位数组用于记录该节点负责哪些槽：

```c
struct clusterNode {
  // ...
  unsigned char slots[16384/8];
  int numslots;
  // ...
};
```

其中`slots`数组以位的形式记录节点占有的槽，`numslots`记录节点占有的槽的数量。

### 传播节点的1槽指派信息

节点会将自己的槽指派信息发送给集群里的其他节点。其他节点收到槽指派信息会同步更新到自身对应的结构体中。

### 记录集群所有槽的指派信息

每个节点的`clusterState`结构体还会有一个长度为16384的指针数组，其指向每个槽所对应的`clusterNode`结构体：

```c
typedef struct clusterState {
  // ...
  clusterNode *slots[16384];
  // ...
} clusterState;
```

### CLUSTER ADDSLOTS的实现

知道了上述的结构之后，其实现就很简单了：

+ 判断是否出现槽冲突（即待分配的槽已经被占用了）
+ 更新本地数据结构
+ 通过消息通知其他节点

> 除非保证集群命令串行，否则当多个冲突的ADDSLOTS命令同时执行的时候就会出现竞争条件。
>
> 到时候看看源码是怎么解决的吧

## 在集群中执行命令

当集群上线的时候，我们就可以通过集群模式的客户端来给集群发送命令了。

当客户端给一个节点发送命令的时候，节点会先计算命令所涉及的键是否属于当前槽，是的话直接执行，否的话抛出`MOVED`错误。

`MOVED`错误会带有命令里的键所属的槽和槽所在的节点的ip和port，客户端在得到该错误返回的时候会根据返回信息找到正确的节点再发送一次请求。

一个键所属的槽=`CRC16(key) & 16383`

> 集群模式的客户端和普通客户端的区别在于，其会处理MOVED错误，并自动切换节点。而普通客户端在接收到MOVED报错的时候无法处理，只会直接抛出。

> 有个问题是，如果命令涉及到两个不同的键，且这两个键所属的槽不同的话，集群模式怎么处理？
>
> 常见的例子就是有序集合的集合间运算。
>
> 所以还是得看源码。
>
> > 刚好工作就遇到了这个问题，提前看了源码
> >
> > 遇到这种情况Redis会出错。也就是Redis仅会在一个节点中执行命令，如果这条命令中包含部分键不在该节点，节点会读取到空的键（对这个节点而言该键不存在），后续工作都将该键当空处理。
> >
> > 一个折中的方案是使用hashtag。Redis在3.0就引入了这个概念。
> >
> > 即，如果键的格式为`{h}k`的话，其计算所属槽所用的key为`h`而不是整个键
> >
> > 这里的`h`就是hashtag

### 节点数据库的实现

不同于普通模式的Redis服务器，作为节点的服务器只能有0号数据库这一个数据库。

其数据库实现跟普通Redis数据库一样，不过其还多出一个跳表来存键和其所在槽的映射关系：

```c
typedef struct clusterState {
  // ...
  zskiplist *slots_to_keys;
  // ...
} clusterState;
```

在这个跳表里，键名作为key，其所对应的槽的下标作为分值。

该结构可以用于查询每个槽中有多少键，以用于重新分片。

## 重新分片

Redis支持将槽从源节点重新分配给目标节点。这个过程称为重新分片。

其操作需要依靠Redis的集群管理软件`redis-trib`，具体操作如下：

+ `redis-trib`对目标节点发送`CLUSTER SETSLOT <slot> IMPORTING <source_id>`命令，让目标节点准备好从源节点接收某个槽
+ `redis-trib`对源节点发送`CLUSTER SETSLOT <slot> MIGRATING <target_id>`命令，让源节点准备好将某个槽发送给目标节点
+ `redis-trib`对源节点发送`CLUSTER GETKEYSINSLOT <slot> <count>`命令，从源节点获取槽中最多`count`个键
+ 对于每一个获取到的键，`redis-trib`会发送一条`MIGRATE`命令，让源节点将指定键的数据发送给目标节点
+ 重复以上两步，直到全部迁移完成
+ `redis-trib`向集群中任意一个节点发送`CLUSTER SETSLOT <slot> NODE <target_id>`命令，将槽指派给目标节点。这个消息会通过消息发送广播至整个集群

### ASK错误

槽迁移的过程中，集群仍旧可以执行外部的命令。这样就会遇到一个问题：当命令指定的键所在的槽所在的节点正好在迁移这个槽，并且这个键正好被迁移走了，该怎么办？

这个时候节点就会抛出ASK错误，这可以让客户端重定向至正确的节点（跟MOVED相似）

要知道这个错误怎么实现，就需要先知道迁移指定的命令的原理

### CLUSTER SETSLOT IMPORTING命令

`clusterState`对象中有一个结构用于记录当前节点正在从其他节点导入的槽：

```c
typedef struct clusterState {
  // ...
  clusterNode *importing_slots_from[16384];
  // ...
} clusterState;
```

如果某个槽（假设这个槽的下标为slot）正在迁移至该节点，则该节点中的`clusterState`对象的`import_slots_from[slot]`指针指向源节点的`clusterNode`对象

### CLUSTER SETSLOT MIGRATING命令

同理，`clusterState`对象也有一个结构记录当前节点正在迁移至其他节点的槽：

```c
typedef struct clusterState {
  // ...
  clusterNode *migrating_slots_to[16384];
  // ...
} clusterState;
```

那么ASK错误的实现就很好理解了：查这两个表就知道了。

### ASKING命令

该命令用于开启客户端结构中的`REDIS_ASKING`的flag。

这个flag表示，在节点接收到一条命令时，如果命令中的键不在当前节点，但是其对应的槽正在导入该节点，并且`REDIS_ASKING`的flag在的话，则破例执行一次（执行后关闭该flag）。

当客户端发送命令得到ASK错误的时候，其会先往目标节点发送`ASKING`命令，然后再转发命令，以保证转发时不会接到`MOVED`错误

## 复制和故障转移

Redis集群中的节点可以是单个服务器，也可以是多个主从服务器组成的大节点，其包括主节点和多个从节点，结构跟第15章讲的主从服务器很类似。

### 设置从节点

向一个节点发送`CLUSTER REPLICATE <node_id>`命令，便可以指定该节点成为目标节点的从节点。

接收到命令之后的从节点会依次做如下事情：

+ 修改自己的`clusterNode`结构，将`slaveof`指针指向主节点：

  + ```c
    struct clusterNode {
      // ...
      struct clusterNode *slaveof;
      // ...
    };
    ```

+ 修改自己在`clusterState.myself.flags`中的属性，去掉原本的`REDIS_NODE_MASTER`，改为`REDIS_NODE_SLAVE`。

+ 调用复制代码，相当于从节点向主节点发送`SLAVEOF`命令。具体过程参见第15章

最终该节点成为某个节点的从节点的信息会广播至集群的所有节点，这些节点会修改自己相应的记录，包括每个节点的从节点信息：

```c
struct clusterNode {
  // ...
  int numslaves;
  struct clusterNode **slaves;
  // ...
};
```

### 故障转移

故障转移机制也跟上一章的类似。只不过上一章是由Sentinel集中监控管理，在集群中则是由所有集群节点共同管理。

#### 故障检测

集群中每个节点都会定期向其他节点发送`PING`命令，超时或失败则会将该目标节点标为疑似下线状态。

节点会通过消息来交换其他节点的上下线信息表，以知道其他节点所认为的节点上下线状态。

当集群中超半数节点将某个主节点标记为疑似下线，则会将该节点标记为已下线，并传播`FAIL`消息，收到该消息的节点同样会把该节点标记为已下线。

#### 故障转移

同样的，集群会选举出一个下线的节点的从节点作为新的主节点，其被选举出来之后主要做三步：

+ 自身执行`SLAVEOF no one`，成为主节点
+ 撤销原主节点的槽指派，将槽全部指派到自身
+ 广播`PONG`消息，通知其他节点更新结构信息，和让剩余的从节点进行`SLAVEOF`

#### 选举节点

算法跟上一章选举Sentinel一样，也利用了纪元和竞争投票（基于Raft算法）

## 消息

集群中的节点通过消息收发来进行通信。消息主要有以下五种：

+ MEET：节点接收到客户端的`CLUSTER MEET`命令时，会向目标节点发送MEET消息，请求加入集群
+ PING：节点向其他节点发送PING请求以检测目标节点的上下线状态。通常有两种情况会执行：
  + 节点每隔一秒会随机取5个节点，并选择其中一个最久没发PING消息的节点发送PING消息
  + 当没给某个目标节点发PING消息的时间超过当前节点的`cluster-node-timeout`的一半时，当前节点也会向目标节点发送PING
+ PONG：当节点收到PING或MEET消息时会回应一条PONG信息。或者节点会广播一条PONG信息让集群的其他节点刷新结构信息。
+ FAIL：当当前节点判断某个节点已经下线的时候，会广播关于该节点的FAIL消息，令接收到消息的节点也将其标记为已下线
+ PUBLISH：当节点接收到`PUBLISH`命令时，其在执行命令的同时也会广播一条PUBLISH信息，让集群的其他节点也执行同样的命令

一条消息由消息头和消息正文组成。

### 消息头

消息头包含消息正文和发消息的节点的信息。其结构如下：

```c
typedef struct {
  // 消息的长度
  uint32_t totlen;
  // 消息的类型
  uint32_t type;
  // 消息正文中包含的节点数量（MEET，PING，PONG）会用到
  uint16_t count;
  // 发送者所处的配置纪元
  uint64_t currentEpoch;
  // 发送者所处的配置纪元，或者其主节点所处的配置纪元
  uint64_t configEpoch;
  // 发送者的名字
  char sender[REDIS_CLUSTER_NAMELEN];
  // 发送者的槽指派信息
  unsigned char myslots[REDIS_CLUSTER_SLOTS/8];
  // 主节点的名字，没有则为全零
  char slaveof[REDIS_CLUSTER_NAMELEN];
  // 发送者的端口号
  uint16_t port;
  // 发送者的标识值
  uint16_t flags;
  // 发送者所处集群的状态
  unsigned char state;
  // 消息正文
  union clusterMsgData data;
} clusterMsg;

union clusterMsgData {
  // MEET, PING, PONG消息的正文
  struct {
    clusterMsgDataGossip gossip[1];
  } ping;
  // FAIL消息的正文
  struct {
    clusterMsgDataFail about;
  } fail;
  // PUBLISH消息的正文
  struct {
    clusterMsgDataPublish msg;
  } publish;
};
```

### MEET PING PONG的实现

由于其用的是同一种消息正文，故节点会通过消息头的`type`值来区分命令。

其消息正文为一个`clusterMsgDataGossip`的柔性数组。

> 柔性数组则为，定义一个数组结构，然后等到需要使用的时候再`malloc`需要的空间。
>
> 故定义其长度为1并不会影响其之后使用的长度。

一般发送者会随机选两个自己已知的节点（主从均可）的信息放入数组中。

消息正文结构如下：

```c
typedef struct {
  // 节点的名字
  char nodename[REDIS_CLUSTER_NAMELEN];
  // 最后一次向该节点发送PING消息的时间戳
  uint32_t ping_sent;
  // 最后一次接收该节点PONG消息的时间戳
  uint32_t ping_received;
  
  char ip[16];
  uint16_t port;
  uint16_t flag;
} clusterMsgDataGossip;
```

这些附带的消息可以让接收者认识新节点或更新已认识的节点的信息

### FAIL消息的实现

该消息只需要告知下线节点的名称即可：

```c
typedef struct {
  char nodename[REDIS_CLUSTER_NAMELEN];
} clusterMsgDataFail;
```

### PUBLISH消息的实现

PUBLISH命令的组成如下：

```
PUBLISH <channel> <message>
```

故节点只需要将命令的信息传达到其他节点即可。

故其结构只需要存channel和message的数据：

```c
typedef struct {
  uint32_t channel_len;
  uint32_t message_len;
  // 实际长度为channel_len+message_len，定义为8只是为了对齐其他消息结构
  unsigned char bulk_data[8];
} clusterMsgDataPublish;
```

