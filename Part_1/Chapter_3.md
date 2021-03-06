# 第3章 链表

C语言中并没有链表结构，故Redis自己实现了一个双向链表。

Redis的列表对象的底层就是使用了这种链表结构，这个在之后介绍对象的时候会详细说明。

## 链表节点

首先，Redis定义了链表节点结构体：

```c
typedef struct listNode {
  struct listNode *prev;
  struct listNode *next;
  void *value;
}listNode;
```

其仅包括前置节点指针、后置节点指针、指向节点值的指针。这是一个标准的双向链表节点结构。

> 在C语言中直接使用`struct`定义结构体和用`typedef struct`定义结构体的区别在于，`typedef`关键字给结构体起了一个别名（也就是`}`后面跟着的名字）。这样之后使用这个结构体定义变量就可以直接用其别名而不用带上`struct`关键字了。

## 链表

仅靠链表节点可以完成链表的定义，但操作起来就很不方便了。

套用前面说到的SDS的例子，我们也需要为这个链表扩展一些基本的变量（例如长度）。

故我们需要一个链表结构体来维护整个链表：

```c
typedef struct list {
  listNode *head;
  listNode *tail;
  unsigned long len;
  
  void *(*dup) (void *ptr);
  void (*free) (void *ptr);
  int (*match) (void *ptr, void *key);
}list;
```

结构体的成员包含两种：

+ 定义双向链表结构的成员：指向头节点的指针、指向尾节点的指针、链表长度
+ 定义双向链表结构的行为的函数指针：
  + dup函数指针：复制链表节点，参数为指向待复制的节点的指针，返回值为指向复制后的节点的指针
  + free函数指针：释放链表节点，参数为指向待释放的节点的指针
  + match函数指针：用于比较两个节点的值是否相等，参数为两个待比较的节点的指针，返回值为比较结果

使用函数指针的目的是可以根据不同实现更加方便地定义不同的链表行为，也就是链表行为的多态。