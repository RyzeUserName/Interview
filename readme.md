

参考 https://github.com/AobingJava/JavaFamily

## 1.基础

### 1.集合框架

#### 1.HashMap

------

​	HashMap 是由链表和数组组合而成的基本数据结构 ，k-v 存储，允许null key 和null value

​	![](https://gitee.com/lifutian66/img/raw/master/img/3688928-84c52979b3788ef7.jpg)



​	初始容量  1 << 4  **16**

​	扩容因子 **0.75f**   即 容量 * 扩容因子 为可存储做多数据，超过会扩容 成原来的  **2** 倍  增加时考虑

​	链表长度 > **8** 由 链表转换成 红黑树  增加的时候 考虑

​	链表长度  < **6** 由 红黑树换成 链表转  split的时候 考虑

​	容量 > **64** 链表  转换成  红黑树 



**注意：**

1. 初始化HashMap,但是不使用put（resize），并不会真正初始化HashMap

2. hash方法 return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16)
	
	将高位数据移位到低位进行异或运算,因为这些数字差异主要在高位上，高位和低位取异或保留高位特征值，使其分布更随机
	
	称其为扰动函数
	
3. tableSizeFor n |= n >>> 4  即取最高位是1之后都是1的那个数，然后+1 变成 10000...，2的次幂

4. resize 函数   头插法，尾插法



#### 2.ConcurrentHashMap
------


#### 3.Hashtable

------
正在 HashMap 基础之上方法上增加 synchronized 

#### 4.ArrayList
------
#### 5.Vector
------
#### 6.Enum
------
### 2.并发与多线程
------
#### 1.互斥锁、自旋锁、读写锁、悲观锁、乐观锁

#### 2.CountDownLatch，CyclicBarrier，Semaphore

#### 3.Volatile

#### 4.ThreadLocal

#### 5.Synchronized

#### 6.AQS

### 3.tcp/http

#### 1.io

#### 2.tcp/ip

#### 3.linux 命令

#### 4.http

#### 5.https

#### 6.netty

### 4.jvm

#### 1.结构

#### 2.垃圾回收算法

#### 3.工具/命令

### 5.数据结构与算法

#### 1.排序

#### 2.二叉树

#### 3.B+树

#### 4.红黑树

### 6.数据库

#### 1.mysql

### 7.Redis

#### 1.数据结构

#### 2.底层实现

#### 3.缓存问题

#### 4.分布式锁

### 8.消息队列

#### 1.RocketMQ

#### 2.RabbitMq

#### 3.Kafka

### 9.分布式

#### 1.Zookpeer

#### 2.Raft

### 10.框架

#### 1.spring cloud

#### 2.doubb





