# 第15章 复制

本章讲的是主从服务器的基础：复制

主服务器通过复制将数据传给从服务器，并在之后同步传递写命令，已达到主从服务器数据一致的目的。

在Redis中，只需要通过客户端像准备成为从服务器的服务器发送`SLAVEOF`命令，就可以让该服务器成为目标服务器的从服务器。例如：

```
SLAVEOF 127.0.0.1 6379 
```

## 旧版复制功能

这里的复制功能是指Redis2.8以前的复制功能。其只做两件事情：同步和命令传播

### 同步

客户端像从服务器发送`SLAVEOF`命令之后，从服务器会像主服务器发送`SYNC`命令以请求同步。

同步过程如下：

+ 从服务器向主服务器发送`SYNC`命令
+ 主服务器收到命令之后执行`BGSAVE`命令，并将从现在开始所接收到的写命令用一个缓冲区记录下来
+ 当主服务器的`BGSAVE`执行完毕之后，会将生成的RDB文件传给从服务器。从服务器载入这份RDB文件
+ 主服务器将缓冲区的写命令传给从服务器，从服务器执行这些命令，已达到跟主服务器数据一致的状态
  + 这个过程中主服务器是阻塞的

### 命令传播

在同步完成之后，为了保持主从服务器的数据一致性，当主服务器从客户端收到写命令时，会将该命令发送给从服务器。

### 旧版复制功能的缺陷

假设一种场景：主从服务器在保持连接的过程中断开连接一小会再重连上。此时主从服务器会有小部分数据是不一致的（也就是断线之后主服务器收到的写命令）。

如果用旧版的策略的话，此时需要重新从头到尾同步一遍主从服务器，包括前面许多已经一致的数据。这显然是一个浪费资源的举措。

## 新版复制功能

Redis从2.8开始，采用`PSYNC`命令代替`SYNC`。这个命令具备两种同步模式：

+ 完整重同步：也就是`SYNC`的同步模式
+ 部分重同步：用于处理上述的断线重连的情况，仅同步断线时没同步的指令

### 部分重同步的实现

其实现需要依靠如下三个部分：

+ 复制偏移量
+ 复制积压缓冲区
+ 运行ID

#### 复制偏移量

主从服务器都会维护一个`offset`变量用以表示偏移量，单位为字节。

假设主服务器的`offset`为10119，其某台从服务器的`offset`为10086，则说明有大小为33字节的指令未同步。

主从服务器在执行写命令的时候会维护这个变量。

#### 复制积压缓冲区

主服务器维护一个固定字节的队列，用于做复制积压缓冲区。其会存储最近N个字节的写指令（N为队列长度）。

这样每次从服务器发送`PSYNC`指令之后，主服务器就可以通过从服务器传上来的偏移量和自己的偏移量对比，如果相差在N以内则可以将缓冲区的写指令直接传播过去即可完成同步。

#### 服务器运行ID

每个Redis服务器都会有一个服务器运行ID，其为一个长度为40的16进制字符串。

从服务器连接主服务器的时候，会保存主服务器的运行ID。当从服务器断线重连的时候，从服务器会将自己所存的主服务器运行ID发给重连的主服务器。如果主服务器对比后发现跟自己的运行ID不一致，则强制执行完整重同步。

## PSYNC的实现

`PSYNC`的命令参数为：

```
PSYNC master_run_id offset
```

其有两种情况：

+ 如果从服务器之前没有复制过任何主服务器（或者执行过`SLAVEOF no one`，这相当于将从服务器重置为没有复制过任何主服务器的状态），则发送`PSYNC ? -1`命令，主动请求主服务器进行完整的重同步。
+ 否则，从服务器按照上面的格式，将自己所存的主服务器运行ID和复制偏移量带上。

主服务器收到`PSYNC`则有三种可能的响应情况：

+ 返回`+FULLRESYNC <runid> <offset>`，则说明主服务器要求进行完整重同步，此时从服务器将发送过来的运行ID和偏移量存下来，为之后接收RDB文件做准备。
+ 返回`+CONTINUE`，则说明主服务器要求进行部分重同步，此时从服务器只需要等主服务器发送命令即可。
+ 返回`-ERR`，则说明主服务器不支持`PSYNC`，从服务器使用`SYNC`命令请求进行完整重同步。

当且仅当主服务器运行ID一致，且偏移量差值小于等于队列长度，否则不会执行部分重同步。

## 复制的实现

这里将从从服务器执行`SLAVEOF ip port`命令开始讲整个复制的过程

### 设置主服务器的地址和端口

当客户端向从服务器发送`SLAVEOF`命令之后，从服务器会将命令里的host和port设置进自己的服务器结构体中：

```c
struct redisServer {
  // ...
  char *masterhost;
  int masterport;
  // ...
};
```

之后给客户端返回OK。

也就是说，只有这一步是同步的，其他复制工作都是在执行完这一步之后异步执行的。

### 建立套接字连接

在执行`SLAVEOF`命令之后，从服务器根据记录的host和port，创建连向主服务器的套接字连接，就像客户端连接服务器一样。

连接完成之后，从服务器即成为了主服务器的一个客户端。

### 发送PING命令

为了确保连接的主服务器是有效的，从服务器会向主服务器发送`PING`命令。

如果主服务器返回错误或超时，则从服务器断开连接并重新连接至主服务器。否则继续。

### 身份验证

当且仅当从服务器设置了`masterauth`选项时，从服务器会请求身份验证。其会将配置的密码用`AUTH`命令发送给主服务器。

当且仅当主服务器设置了`requirepass`选项时，主服务器会进行身份验证，接收`AUTH`命令并进行判定。

也就是说：

+ 当从服务器和主服务器均未设置，则忽略这一步
+ 当从服务器和主服务器均设置，则进行身份校验
+ 只有从服务器设置的话，主服务器会返回`no password is set`的错误
+ 只有主服务器设置的话，之后执行命令主服务器会返回`NOAUTH`的错误

> 这里的存疑点是，假设需要身份验证，那之前的PING命令会返回的不应该是PONG，而是NOAUTH。
>
> 不过这种小细节Redis肯定有处理的，到时候留意一下代码即可

### 发送端口信息

之后，从服务器会使用`REPLCONF listening-port <port>`命令向主服务器发送自己的端口。主服务器接收之后会记到从服务器对应的客户端结构体的`slave_listening_port`中。

### 同步

前面的动作基本都是在建立连接和确保连接可用，以及同步配置。而这一步开始真正的复制。

此时从服务器会使用`PSYNC`命令进行同步。过程前面说过所以不再重复。

有一点需要注意的是，此时主服务器会成为从服务器的客户端。其作用为给从服务器发送命令。

### 命令传播

同步完成之后，主从服务器就会进入命令传播阶段。只要有写命令主服务器都会通过连接发往从服务器。

## 心跳检测

在命令传播阶段，从服务器每隔一秒会给主服务器发送心跳检测命令：

```
REPLCONF ACK <offset>
```

此举有三个目的：

+ 检查网络连接
  + 同时更新主服务器上的延迟时间
+ 检测命令丢失
  + 通过对比offset可以实现
+ 辅助实现min-slaves选项

### min-slaves选项

Redis有两个选项可以防止主服务器在不安全的情况下执行命令：

```
min-slaves-to-write 3
min-slaves-max-lag 10
```

那么当主服务器所拥有的从服务器数量少于`min-slaves-to-write`个，或者所有的从服务器的延迟都大于等于`min-slaves-max-lag`秒时，主服务器拒绝执行写命令。