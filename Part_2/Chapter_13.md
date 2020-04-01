# 第13章 客户端

Redis服务器在与客户端进行连接的时候会针对连接的客户端创建一个`redisClient`结构，用于存储该客户端的状态。`redisServer`结构包含了一个客户端结构列表：

```c
struct redisServer {
  // ...
  list *clients;
  // ...
};
```

`clients`指向的列表中的元素为一个`redisClient`对象。

## 客户端属性

`redisClient`结构包含许多跟客户端状态相关的成员。这里只讲部分主要属性。

### 套接字描述符

客户端结构体记录了当前正在使用的与客户端通信的套接字描述符：

```c
typedef struct redisClient {
  // ...
  int fd;
  // ...
} redisClient;
```

其存在两种可能：

+ fd=-1：表示其是个伪客户端，伪客户端用于处理来自AOF文件或者Lua脚本的请求。
+ 否则：该套接字便是记录与客户端连接的套接字

### 名字

Redis可以通过`CLIENT setname`命令给客户端设置名字。名字作为一个Redis对象存在客户端结构体中：

```c
typedef struct redisClient {
  // ...
  robj *name;
  // ...
} redisClient;
```

当客户端没有名字的时候，其值为NULL；否则指向一个字符串对象。

### 标志

客户端的标志属性记录客户端的角色和状态等信息，例如：

+ 该客户端为一个版本低于Redis2.8的从服务器（`REDIS_PRE_PSYNC`）
+ 该客户端正被`BRPOP`、`BLPOP`阻塞（`REDIS_BLOCKED`）
+ 该客户端的输出缓冲区大小超出了服务器允许的范围（`REDIS_CLOSE_ASP`）

等等。。。

```c
typedef struct redisClient {
  // ...
  int flags;
  // ...
} redisClient;
```

这个值可以等于某一个标志常量，或者多个标志常量的位或。

> 这些标志常量估计都只有一位是1，且各不相同。

### 输入缓冲区

客户端结构体维护一个输入缓冲区用于存放还没处理的命令：

```c
typedef struct redisClient {
  // ...
  sds querybuf;
  // ...
} redisClient;
```

它会将要执行的命令转换成存AOF文件的格式（以`SET key value`为例）：

```
*3\r\n$3\r\nSET\r\n$3\r\nkey\r\n$5\r\nvalue\r\n
```

再塞入缓冲区中。

缓冲区大小最多1GB，否则会被服务器强行关闭。

### 命令与命令参数

服务器会从客户端的缓冲区中取出单条命令，存入命令参数区：

```c
typedef struct redisClient {
  // ...
  robj **argv;
  int argc;
  // ...
} redisClient;
```

`argv`为一个字符串对象数组，存储命令的各个用空格隔开的单词。而`argc`则是数组的元素个数。

### 命令的实现函数

Redis服务器在读到命令之后，就会去命令表里找对应的实现函数。

命令表是一个字典结构，其键为字符串对象，也就是命令的字符串，值为对应的实现函数结构体`redisCommand`。

Redis客户端结构体存了一个指向`redisCommand`结构体的指针：

```c
typedef struct redisClient {
  // ...
  struct redisCommand *cmd;
  // ...
} redisClient;
```

当服务器读到命令并将命令存入`argv`之后，就会根据`argv[0]`的值去命令表里找到对应的`redisCommand`结构体，并让`cmd`指针指向它。之后就可以执行命令了。

### 输出缓冲区

Redis在执行完指令之后会将返回值放到输出缓冲区中。

客户端结构体的输出缓冲区有两个：

+ 定长缓冲区`buf`（包括记录其使用的字节数的变量`bufpos`）
+ 变长缓冲区列表`reply`，列表元素为字符串对象

```c
typedef struct redisClient {
  // ...
  char buf[REDIS_REPLY_CHUNK_BYTES];
  int bufpos;
  
  list *reply;
  // ...
} redisClient;
```

当定长区足够用的时候就会存入定长区，否则才去使用缓冲区列表

### 身份验证

客户端结构体存了一个变量用于表示是否通过身份验证：

```c
typedef struct redisClient {
  // ...
  int authenticated;
  // ...
} redisClient;
```

当其为1时表示已验证，否则为未验证。

不过如果服务端不开启验证的话，即使是0也表示已验证。

### 时间

客户端会存几个时间戳：

```c
typedef struct redisClient {
  // ...
  time_t ctime;
  time_t lastinteraction;
  time_t obuf_soft_limit_reached_time;
  // ...
} redisClient;
```

+ `ctime`：客户端创建时间
+ `lastinteraction`：客户端与服务端最后一次互动（发数据/收数据均可）的时间
+ `obuf_soft_limit_reached_time`：输出缓冲区第一次到达软性限制的时间，后面会提到

## 客户端的创建和关闭

### 创建普通客户端

当客户端通过`connect`函数发起对服务端的连接时，服务端调用连接事件处理器，创建一个新的客户端结构体，放入服务端的客户端结构体列表里。

### 关闭普通客户端

Redis服务器会有如下几种情况会关闭客户端：

+ 客户端进程退出/杀死
+ 客户端发送不符合协议的数据
+ 客户端发送`CLIENT KILL`命令
+ 服务器设置了`timeout`，且客户端空转时间超过`timeout`
  + 有一种例外就是，客户端是主服务器，且当前服务器正被阻塞或正在订阅，则无视`timeout`
+ 输入缓冲区溢出
+ 输出缓冲区溢出

虽然客户端有可变长的输出缓冲区，但为了防止缓冲区耗尽内存资源，故会有两种方式同时设置其大小：

+ 硬性限制：设置一个大小，当输出缓冲区超出这个大小则立刻关闭
+ 软性限制，设置一个大小，当输出缓冲区超出这个大小限制的时候，服务器会记下时间（存在之前提到的`obuf_soft_limit_reached_time`里）并一直监视，如果在一段时间内一直都是超出软性限制则关闭

一般而言硬性限制的大小会比软性限制的大

### 伪客户端

服务器会有在本地创建伪客户端来模拟请求响应的需求，一共有两种情况：

+ 初始化服务器时，如果有AOF持久化，则创建一个伪客户端用于发送AOF文件命令
+ 服务器本身会创建一个常驻的伪客户端用于发送从Lua脚本中解析出来的命令

