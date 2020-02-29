# Redis设计与实现

得益于疫情期间可以在家办公，多出了些许空闲时间。

其中一半时间用于安排玩更多的游戏之外，另一半的时间就是用来看书了。

可能是大学学习路线的影响，在一开始接触某个技术的时候总是会想要去弄清它的原理。

因为对我来说，只有知道了其原理，我才能够对其性能有一个踏实的把握，而不是靠玄学Rush。

不过工作中接触的东西都十分复杂（大型网络框架、MySQL这种复杂的数据库、etc），随便拿一个的原理我都不知道要摸多久才能搞懂。所以就挑了Redis这个看起来比较软的柿子先试试手。

虽然看了《Redis实战》之后发现这其实好像也没软到哪去。。其涉及到事务、发布订阅、多机、持久化等情况一点没比MySQL简单。

只能硬着头皮试试看了。

这里使用的是黄建宏所著的《Redis设计与实现》

## 目标

### 基础目标

结合自己的年度计划，我需要在4月份结束之前看完这本书，也就是**2020年4月30日**之前搞定。

从3月1号开始算起，这一共有43个工作日和17个周末和1个清明假期。

这里规定工作日学习1份时间，则周末学习2份时间（周末还是放松比较好hhhh），假期学习1份时间（假期更要放松了），则一共有78份时间。

本书除了介绍性的第一章之外，一共有23章，则每章占用3份时间的情况下，还剩9份时间。

将这9份时间补贴到一些较长的章节之后，就可以拟出大概的DDL表了。

### 进阶目标

本书的进阶目标不再是时间的缩短，而是任务的加重。

即为将书中的知识进行实践，也就是啃Redis最新版本源码。

这里使用的源码是最新的（截止至2020.02.26）的Redis 5.0.7版本：https://github.com/antirez/redis/tree/5.0.7

目前的打算是：

+ 先不看源码读一遍，大致了解Redis的骨架并做笔记
+ 然后带着源码再读一遍，详细了解Redis源码

## 进度表

### 第一部分 数据结构与对象

+ [x] 第2章 简单动态字符串
  + DDL：2020.03.02
  + 完成时间：2020.02.26
+ [x] 第3章 链表
  + DDL：2020.03.05
  + 完成时间：2020.02.29
+ [ ] 第4章 字典
  + DDL：2020.03.07
+ [ ] 第5章 跳跃表
  + DDL：2020.03.09
+ [ ] 第6章 整数集合
  + DDL：2020.03.12
+ [ ] 第7章 压缩列表
  + DDL：2020.03.14
+ [ ] 第8章 对象
  + DDL：2020.03.17

### 第二部分 单机数据库的实现

+ [ ] 第9章 数据库
  + DDL：2020.03.21
+ [ ] 第10章 RDB持久化
  + DDL：2020.03.23
+ [ ] 第11章 AOF持久化
  + DDL：2020.03.26
+ [ ] 第12章 事件
  + DDL：2020.03.28
+ [ ] 第13章 客户端
  + DDL：2020.03.30
+ [ ] 第14章 服务器
  + DDL：2020.04.03

### 第三部分 多机数据库的实现

+ [ ] 第15章 复制
  + DDL：2020.04.05
+ [ ] 第16章 Sentinel
  + DDL：2020.04.09
+ [ ] 第17章 集群
  + DDL：2020.04.12

### 第四部分 独立功能的实现

+ [ ] 第18章 发布与订阅
  + DDL：2020.04.15
+ [ ] 第19章 事务
  + DDL：2020.04.18
+ [ ] 第20章 Lua脚本
  + DDL：2020.04.20
+ [ ] 第21章 排序
  + DDL：2020.04.24
+ [ ] 第22章 二进制位数组
  + DDL：2020.04.26
+ [ ] 第23章 慢查询日志
  + DDL：2020.04.28
+ [ ] 第24章 监视器
  + DDL：2020.04.30