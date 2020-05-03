# 第16章 Sentinel

Sentinel（哨兵）为Redis的高可用性解决方案。其基本思想为使用特殊的服务器监视主从服务器群。其需要做的事情包括：

+ 一旦有主服务器挂掉了立刻选出一台从服务器作为新的主服务器，并重新建立服务器的主从关系。

+ 当挂掉的原主服务器恢复时，将该服务器作为新的主服务器的从服务器。

## 初始化Sentinel

Sentinel本质上为特殊的Redis服务器，故可以使用启动Redis服务器的方式启动Sentinel：

```
redis-server /path/sentinel.conf --sentinel
```

或者使用`redis-sentinel`启动：

```
redis-sentinel /path/sentinel.conf
```

### 初始化服务器

由于Sentinel本质是Redis服务器，故其第一步就是初始化一个Redis服务器。

不过具体的步骤会跟普通的Redis服务器不完全相同，包括：

+ Sentinel不需要载入RDB或AOF文件，更不需要使用数据库
+ Sentinel只支持少量命令，包括`INFO`，`PING`，`SENTINEL`，和订阅频道/取消订阅等命令

### 使用Sentinel专用代码

其第二步是将部分普通的Redis服务器代码替换为Sentinel专用代码。例如：

+ Sentinel使用`sentinel.c/REDIS_SENTINEL_PORT`常量作为服务器端口，而不是`redis.h/REDIS_SERVERPORT`
+ Sentinel使用`sentinel.c/sentinelcmds`作为服务器的命令表，而不是`redis.c/redisCommandTable`。其使用的命令函数也是不同的。

> 不太明白为什么在初始化之后才替换代码，我的理解是初始化之后很多代码都已经编译了才对。
>
> 我想这更应该算是更换配置？只能之后去源码里确认细节了

### 初始化Sentinel状态

接着初始化一个Sentinel状态结构体：

```c
struct sentinelState {
  // 当前纪元。故障转移用
  uint64_t current_epoch;
  // 保存所有被监视的主服务器信息。键为主服务器名，值为一个sentinelRedisInstance指针
  // 这里不保存从服务器信息，从服务器由主服务器结构体保存
  dict *masters;
  // 是否使用tilt模式
  int tilt;
  // 目前正在跑的脚本数量
  int running_scripts;
  // 进入tilt模式的时间
  mstime_t tilt_start_time;
  // 最后一次执行时间处理器的时间
  mstime_t previous_time;
  // 需要执行的脚本队列
  list *scripts_queue;
} sentinel;
```

### 初始化Sentinel状态的masters属性

接着就是根据配置文件初始化masters字典了。

其结构体`sentinelRedisInstance`如下：

```c
typedef struct sentinelRedisInstance {
  // 标记值，记录实例的类型（主服务器还是从服务器）和状态
  int flags;
  // 实例名，一般为ip:port
  char *name;
  // 实例运行id
  char *runid;
  // 配置纪元。故障转移用
  uint64_t config_epoch;
  // 实例的地址
  sentinelAddr *addr;
  // 实例无响应多少毫秒后才会被判定为主观下线
  mstime_t down_after_period;
  // 判断这个实例为客观下线所需的支持投票数量
  int quorum;
  // 在执行故障转移操作时，可以同时对新的主服务器进行同步的从服务器数量
  int parallel_syncs;
  // 刷新故障迁移的最大时限
  mstime_t failover_timeout;
  // 从服务器字典
  // 该字典的键为服务器名，值为一个sentinelRedisInstance指针
  dict *slaves;
  // Sentinel字典，格式同上
  // 保存所有监听自己的Sentinel的信息
  dict *sentinels;
  // ...
} sentinelRedisInstance;

typedef struct sentinelAddr {
  char *ip;
  int port;
} sentinelAddr;
```

### 创建连向主服务器的网络连接

最后一步是创建连接。对于每个主服务器需要创建两个连接：一个用于发送命令，一个用于订阅`__sentinel__:hello`频道。

出于对订阅频道功能的高可用要求（尽可能不要漏掉信息），故订阅频道另开一个连接，而不是跟发送命令共用一个连接。

## Sentinel日常任务

在一般情况下，Sentinel会做如下几个事情

### 获取主服务器信息

Sentinel每十秒会向监视的主服务器发送INFO命令，以获取其最新信息并更新到自己的字典中去。这其中就包括发现主服务器所拥有的从服务器。

### 获取从服务器信息

当Sentinel通过获取主服务器信息发现从服务器的时候，除了将从服务器的实例添加到主服务器实例的字典中之外，还会创建一个命令连接和一个订阅连接。之后同样会每隔十秒发送一遍INFO命令来获取服务器信息并更新实例。

### 向主服务器和从服务器发送消息

Sentinel每隔两秒通过命令连接向其所有被监视的主从服务器使用发布命令：

```
PUBLISH __sentinel__:hello "<s_ip>,<s_port>,<s_runid>,<s_epoch>,<m_name>,<m_ip>,<m_port>,<m_epoch>"
```

其中s开头的是自己的信息，m开头的是其目标服务器的主服务器的信息（如果自身本身就是主服务器则为自身的信息）。

### 接收来自主从服务器的频道信息

Sentinel通过订阅频道可以接收到其他Sentinel发送给自己所监视的主从服务器的信息。

Sentinel借此可以发现其他Sentinel，并将其他Sentinel的信息保存或更新到对应主服务器实例的`sentinels`字典中。

同时，创建连向其他Sentinel的命令连接。

## Sentinel检测故障和故障转移

Sentinel通过一系列流程来判定某个服务器是否下线，以及如何进行转移操作

### 检测主观下线状态

Sentinel每隔一秒会向所有命令连接发送PING命令，已检测对方是否存活。

而当命令接收方连续`down_after_period`毫秒无响应，则会被判定为主观下线，此时设置其`flags`值以标记其主观下线。

主观下线状态适用于Sentinel监视的主服务器、从服务器、Sentinel。

### 检测客观下线状态

当Sentinel判断一个主服务器为主观下线时，其就会向其他Sentinel发送如下命令：

```
SENTINEL is-master-down-by-addr <ip> <port> <current_epoch> <runid>
```

> 这条命令以及其返回中的epoch和runid先无视，这是之后会用到的。现阶段其只可能是0和`*`

在runid为`*`的情况下，该命令表示向接收命令的Sentinel询问目标主服务器是否下线。

收到该命令的Sentinel查询自己对应的主服务器实例是否是主观下线，并进行如下答复：

```
1) <down_state>
2) <leader_runid>
3) <leader_epoch>
```

第一个参数表示该主服务器在当前Sentinel的记录里是否主观下线（是为1，否为0）。

而当Sentinel接收到大于等于`quorum`个肯定回复时，Sentinel便将该主服务器实例设置为客观下线。

### 选举领头Sentinel

接下来，需要从所有认为主服务器客观下线的Sentinel中选出一个领头Sentinel来执行故障转移。

选举算法类似Raft算法，其使用的仍旧是上面的命令，不过会用上epoch和runid参数。

过程如下：

+ 所有在线的Sentinel都有选举权，其会认定一个其他Sentinel为其领头Sentinel，也就是局部领头。
+ 所有认为主服务器客观下线的Sentinel都会发送`SENTINEL is-master-down-by-addr`请求，带上当前epoch和自己的runid来要求其他Sentinel将自己选为领头。
  + epoch为纪元。每轮选举之后所有Sentinel的epoch都会加一。换句话说就是选举轮数计数器。
+ 当一个Sentinel收到带runid的`SENTINEL is-master-down-by-addr`请求时，如果其没有认定过领头，则将收到的runid认定为领头。
  + 在此之前会先根据epoch确认是否是当前纪元的选举，丢弃掉过期的请求。
  + 不论成功与否，Sentinel都会将自己的领头runid和纪元返回过去
+ 而Sentinel收到其他Sentinel的返回时，就可以知道自己得到多少选票。当其得到超过半数选票时自动成为领头Sentinel。
  + 在此之前仍旧会根据epoch过滤掉过期的返回
  + 如果一轮选举下来没有出现领头Sentinel·，则各个Sentinel在一段时间后重新选举。直到选出为止。

### 故障转移

故障转移分为三步：

+ 选出合适的从服务器作为新的主服务器
+ 将其他从服务器连接到新的主服务器
+ 等待原主服务器上线，并将其设置为新的主服务器的从服务器

#### 选出新的主服务器

Sentinel会根据如下标准依次筛选现有从服务器并选出候选主服务器

+ 删除下线的从服务器
+ 删除最近5秒没有回复过领头Sentinel的INFO命令的从服务器
+ 删除所有和已下线的主服务器连接断开超过`down-after-milliseconds * 10`毫秒的服务器
  + `down-after-milliseconds`指定了判断主服务器下线所需的时间。如果连接断开超过其10倍，说明这个从服务器的数据会有一定的延迟。
+ 根据从服务器的优先级排序，取最高优先级的
  + 这个优先级在哪，怎么获得，好像没说。到时候看源码
+ 如果还有多个，则选取其中偏移量最大的
+ 如果还有多个，则选runid最小的

之后Sentinel会使用如下手段将该服务器变为主服务器：

+ 向其发送`SLAVEOF no one`命令
+ 每隔一秒向其发送`INFO`命令，读取返回中的`role`字段来判定其是否成功转变为主服务器

如果成功转变，则进行下一步

#### 修改从服务器的复制目标

这一步很简单，给剩余的所有从服务器发送新的`SLAVEOF`命令即可。

#### 将旧的主服务器改为从服务器

当旧的主服务器上线时，给其发送`SLAVEOF`命令即可。

> 这里是定时监视其状态（用PING、INFO等），还是不断发送SLAVEOF直到成功为止，并没有说。到时候留意源码。