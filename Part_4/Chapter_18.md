# 第18章 发布与订阅

Redis提供发布订阅模式。

客户端可以在某台服务器上订阅某个特定频道，或者一次性订阅所有名字满足特定模式（pattern）的频道。

而当服务器收到发布到这个频道上的消息的时候，其就会将这条消息推送到所有订阅这个频道的客户端上。

## 频道的订阅与退订

Redis在`redisServer`对象中维护一个字典用于记录频道的订阅关系：

```c
struct redisServer {
  // ...
  dict *pubsub_channels;
  // ...
};
```

其键为频道名，值为一个链表，存储所有订阅这个频道的客户端。

> 具体存了客户端的什么，书里没说。个人倾向于存一个指向对应`redisClient`对象的指针。
>
> 不过还是得看源码

那么频道的订阅和退订本质上就是在这个字典上添加和删除。

当进行频道订阅时，如果已经存在该频道则直接尾部添加，否则新建键值对和空链表再尾部添加。

当进行频道退订时，直接遍历对应链表找到对应客户端进行删除，如果此时该频道链表为空则删除键值对。

在Redis中，使用`SUBSCRIBE`和`UNSUBSCRIBE`来订阅和退订频道。

## 模式的订阅和退订

前面说过，Redis支持使用一个pattern来订阅多个频道。

例如使用`new.[ie]t`来订阅`new.it`和`new.et`两个频道。

在Redis中，其用另一个数据结构来维护模式订阅：

```c
struct redisServer {
  // ...
  list *pubsub_patterns;
  // ...
};
```

链表中的元素为一个`pubsubPattern`对象：

```c
typedef struct pubsubPattern {
  redisClient *client;
  robj *pattern;
} pubsubPattern;
```

其存放一个客户端指针和一个表示模式的字符串。

换言之，即使存在两个客户端订阅了相同的模式，其在底层存储中也是两个无关联的链表节点。

那么模式的订阅和退订本质上就是链表的添加节点和删除节点。

在Redis中，使用`PSUBSCRIBE`和`PUNSUBSCRIBE`来订阅和退订模式。

## 发送消息

Redis使用`PUBLISH`命令让服务器进行消息推送：

```
PUBLISH <channel> <message>
```

其实现步骤分两步：

+ 推送给频道订阅者
+ 推送给模式订阅者

### 推送给频道订阅者

这一步比较简单，只需要在字典中找到频道名对应的链表，然后扫一遍链表并逐个发送消息即可。

### 推送给模式订阅者

这一步比较麻烦，其需要将整个模式订阅链表扫一遍，并且对每个节点都进行一个模式匹配检查，匹配了就推送给对应的客户端。

## 查看订阅信息

`PUBSUB`命令可以查看一些订阅相关的信息

### PUBSUB CHANNELS

该命令返回服务器当前被订阅的所有频道，其支持附带一个pattern参数用于返回所有能匹配pattern的频道：

```
PUBSUB CHANNELS	// 返回所有频道名
PUBSUB CHANNELS [pattern]	// 返回只跟pattern匹配的频道名
```

实现原理为遍历整个`pubsub_channels`字典，取出满足条件的key

### PUBSUB NUMSUB

该命令接受任意多个频道名，返回频道的订阅者数量：

```
PUBSUB NUMSUB [channel_1 channel_2 ...]
```

实现原理为直接访问`pubsub_channels`，取对应的链表长度

### PUBSUB NUMPAT

该命令返回进行模式订阅的客户端的数量。

实现原理为直接取`pubsub_patterns`的长度