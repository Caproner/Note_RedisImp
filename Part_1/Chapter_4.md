# 第4章 字典

字典泛指可以存储多个键值对，并根据键查询值的数据结构。C语言并没有这种结构，故Redis自己实现了一个。

在Redis中，字典的用途十分广泛。数据库键就是用字典维护的，哈希表的底层也是这玩意。

字典有多种实现方法，Redis使用了哈希的实现方法。

## 字典的实现

字典的实现相对复杂。

### 哈希表

哈希表的结构如下：

```c
typedef struct dictht {
  dictEntry **table;
  unsigned long size;
  unsigned long sizemask;
  unsigned long used;
}dictht;
```

首先是第一个成员：哈希表的本体`table`。`table`指针指向一个`dictEntry*`的数组，这个的数组的下标即为哈希索引值，值为一个哈希节点单链表。

于是后面的三个值就很好懂了：

+ `size`：`table`指向的数组的大小，其值永远为`2^n`
+ `sizemark`：哈希表掩码，永远等于`size-1`
+ `used`：哈希表的节点数（数组的所有链表的所有节点的和）

### 哈希表节点

哈希表节点的结构如下：

```c
typedef struct dictEntry {
  void *key;
  union {
    void *val;
    uint64_t u64;
    int64_t s64;
  } v;
  struct dictEntry *next;
}dictEntry;
```

首先，一个哈希节点必然包括键值：

+ `key`：指向键的指针
+ `v`：键所对应的值，其可以是一个泛型指针，也可以是一个64位的无符号或带符号的整型

其次，通过上面可以知道，哈希表存储的是多个哈希节点的单链表，故其需要一个指向下一个节点的指针`next`

### 字典

字典的结构除了哈希表之外，还存储了指向字典基本操作的函数指针（就像上一章的链表一样）：

```c
typedef struct dict {
  dictType *type;
  void *privdata;
  dictht ht[2];
  int rehashidx;
}dict;
```

其中比较好理解的就是字典所用的哈希表本体`ht`。默认的哈希表是`ht[0]`，`ht[1]`是用于进行rehash的时候的缓冲区。

顺带下去的就是整型成员`rehashidx`，当字典没有在进行rehash的时候这个值为-1，其他的之后讨论rehash的时候会提到。

接着就是定义其行为的`*type`。不同于链表将函数指针放进母结构，字典是新建了一个`dictType`的子结构专门存放字典基础行为的函数指针。其定义如下：

```c
typedef struct dictType {
  unsigned int (*hashFunction) (const void *key);
  void *(*keyDup) (void *privdata, const void *key);
  void *(*valDup) (void *privdata, const void *obj);
  int (*keyCompare) (void *privdata, const void *key1, const void *key2);
  void (*keyDestructor) (void *privdata, const void *key);
  void (*valDestructor) (void *privdata, const void *obj);
}dictType;
```

其中包括的函数有：

+ `hashFunction`：计算键的哈希值
+ `keyDup`：复制键
+ `valDup`：复制值
+ `keyCompare`：比较两个键
+ `keyDestructor`：销毁键
+ `valDestructor`：销毁值

注意到除了计算哈希值的函数之外都带有`privdata`参数。这是用于传入字典中的`privdata`以便于进行一些更加定制化的操作（可以理解为多一个可选参数）

## 哈希算法

一个哈希节点要放入表中的必经之路就是计算哈希索引。

Redis将计算哈希索引这个步骤分为两步：

+ 调用`hashFunction`函数，计算key的哈希值
  + Redis一般会使用MurmurHash2算法计算哈希值
+ 将这个哈希值与`sizemark`位于，得到索引
  + 这也是`size`必须为`2^n`的原因，这样哈希值位于之后便等可能地落在`0`到`size-1`之间

得到索引之后就可以直接找到对应的数组位置，然后头部插入数组位置对应的链表即可。

## 解决键冲突

显然，这里使用的是拉链法解决的。

> 但是这样就有一个问题：遇到完全相同key的时候怎么处理？
>
> 按照一般拉链法的实现的话，哈希后找到对应的链之后需要扫一遍链，确保没有重复（重复的话就替换或无视，根据Redis的表现来看应该是替换），然后再插入。
>
> 但书里仅提到【为了效率考虑，直接头部插入】，也就是说其有可能忽视掉旧的相同键。
>
> 如果真的是这样的话那就会有三个问题：
>
> + 旧的节点永远不会用到，这会产生冗余
> + 删除key对应的值的时候，旧节点会不会跟着删除
> + 后续rehash的时候，怎么保证新节点永远在旧节点之前
>
> 关于这些需要在后续读源码的时候关注

## rehash

当哈希表的节点过多的时候需要扩展表，节点过少的时候需要收缩表，这两者都会涉及到rehash操作。

之前提到的，哈希表的本质是存储拉链的数组，这意味着其链数是固定的，rehash的目的就是改变数组大小。

### 新的size取值

分两种情况：

+ 如果要扩展表，则新的size为第一个大于等于`ht[0].used*2`的`2^n`
+ 如果要收缩表，则新的size为第一个大于等于`ht[0].used`的`2^n`

### 表的迁移

这个时候就需要用到之前提到的缓冲区`ht[1]`了。其整个过程包括：

+ 计算新的size，并以此初始化`ht[1]`
+ 对于所有的`ht[0]`中的节点，计算新的哈希值并放入`ht[1]`
+ 销毁`ht[0]`，让`ht[0]`指向`ht[1]`指向的空间，并给`ht[1]`新建一个空的哈希表

> 其实这个过程中的第二步是存疑的。出于如下原因：
>
> + 新旧哈希表的大小是成倍数关系的
> + 新旧哈希表的哈希计算函数不变，仅仅是`sizemark`变了
>
> 这意味着其可以利用倍数关系做很多优化（例如说收缩的时候就不用一个一个扫了，而是直接把整个链表搬过去），但书里没提到。
>
> 关于这些需要在后续读源码的时候关注

### 何时收缩/扩展

首先，我们定义一个负载因子`load_factor`，这个因子用于评定哈希表的饱和度。

其计算方式为：`load_factor = ht[0].used / ht[0].size`

当以下条件的其中一个被满足时，进行扩展操作：

+ 如果服务器目前没有进行`BGSAVE`或`BGREWRITEAOF`，且`load_factor`大于等于1时
+ 如果服务器目前正在进行`BGSAVE`或`BGREWRITEAOF`，且`load_factor`大于等于5时

> 进行持久化操作的时候放宽扩展条件的目的是避免持久化过程中触发rehash
>
> 这是因为rehash操作和持久化操作一样会进行大规模写入，由于许多操作系统都采用写时复制的方式，这意味着只有当fork出来的进程进行写操作的时候才会真正获得父进程的空间副本。rehash显然会触发这个机制使得其会独立占用一块内存较长时间。从避免内存占用的角度讲需要避免在持久化过程中争夺内存。

当`load_factor`小于0.1时，进行收缩操作

### 渐进式rehash

当哈希表足够大的时候，一次rehash可以让哈希表长时间被占用。这个时候去读写该哈希表的时候就得等到rehash完成，从而要等很长时间。这显然是用户无法忍受的。

故我们需要将rehash的步骤拆一下，一步一步来，让这个过程中的其他读写操作可以穿插在这之间进行。这就是渐进式rehash。

Redis会根据`ht[0].table`指向的数组的下标拆分步骤，一次只搬运一个下标对应的链表。

#### rehashidx

这个时候`rehashidx`就派上用场了。其会记录【当前哈希到哪个下标了】。

简单地讲，假设当前数组下标为0，1，2，3的链表已经完成搬运，那么`rehashidx`的值就是3。

#### rehash期间的读写操作

对于读操作而言，字典除了读`ht[0]`之外，还需要多读一个`ht[1]`。

对于写操作而言，如果是无需找到对应键的操作（例如新增），则默认写进`ht[1]`。