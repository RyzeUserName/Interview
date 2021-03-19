

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

4. resize 函数   新的位置 是原位置或者原位置+旧容量，尾插法（头插法有问题）

   尾插法在resize 扩容时，1234 可能会变成 124或者1234.. 但是不会改变数据之前的前后顺序，头插则会421或者4321..，

   改变前后顺序，在并发会导致循环引用或者丢值
   
   1.8尾插法也会在并发情况下，产生值覆盖 的情况



#### 2.ConcurrentHashMap
------

 在HashMap 的基础上 增加了可并发能力，效率比hashtable高，没hashmap高



1.7版本 

- 采用的是分段锁，Segment， 继承  ReentrantLock ，每个桶都是一个  Segment ，也就是说其支持并发数取决于其容量
- putVal 中  tryLock() ? null ：scanAndLockForPut （自旋获取超过MAX_SCAN_RETRIES 之后 阻塞获取）

1.8版本

​	抛弃Segment,采用 cas+ synchronized ，cas先设置，否则自旋，如果都不满足 synchronized 设值



**注意：**

​	1.不允许key || value为null    异常：NullPointerException （无法判断 值就是null 还是key还该map中不存在）

#### 3.Hashtable

------
在 HashMap 基础之上方法上增加 synchronized  

初始化为 11

扩容规则为当前容量*2  + 1

遍历采用的 是  Enumeration 

#### 4.ArrayList
------

数组实现的 list 增删慢，查询快

初始长度为 10

无参数的初始化 数组是 {} 只有在 add才会 初始化

有参构造 虽然数组初始化了，但是 set方法中 rangeCheck 会检测 设置的index>=size 抛出异常

扩容为 old+ old/2

#### 5.Vector 

------
Vector 是 ArrayList 所有方法加上 synchronized ，只能保证但操作的原子性，多操作需要加锁

#### 6.Enum

------
枚举 自 jdk 1.5之后，说白了就是为了替代 多意义常量变量的存在

枚举的compareTo() 方法 返回 ordinal 的差值

ordinal 返回在枚举中的顺序下标 从0 开始的

原理：

​	其实就是 继承**java.lang.Enum** 类的抽象类，枚举值变成了 **public static final**的属性，其实现为内部类（继承上面抽象类）

​    所有的枚举变量都是通过静态代码块进行初始化，在类加载期间完成(饿汉式的单例)

​	**通过把clone、readObject、writeObject这三个方法定义为final，保证了每个枚举类型及枚举常量都是不可变的**，也就是说 可以用枚举实现线程安全的单例

### 2.并发与多线程

------
#### 1.互斥锁、自旋锁、读写锁、悲观锁、乐观锁

1. **互斥锁**

   互斥锁是独占锁，成功获取 继续下面流程，失败的话释放cpu，进入阻塞状态，等待获取锁释放（这个过程是操作系统内核实现的）

   失败的话线程从 

   ​		用户态-->内核态  等待内核将其唤醒 再由  内核态 -->用户态 

   ​		运行--> 阻塞         等待内核将其唤醒 再由   阻塞--> 就绪

   **开销**： 两次上下文切换,同一个进程内，**虚拟内存**共享，只需要切换线程私有的属性即可，耗时大概在几十纳秒到几微秒之间

   ​			**注意：**被锁住的代码执行时间过短，就不要用互斥锁，两次上下文切换耗时可能超过代码执行时长，改用**自旋锁**

   

   ![](https://mmbiz.qpic.cn/mmbiz_png/J0g14CUwaZflJahXjfiaG4OvTA9DA2UibzKwEMKCNn2DoRsgWyvZsfzPARRpvfdc3ywicDNAmVrIFE6icduenBnxgw/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

   

2. **自旋锁**

   **自旋锁**加锁失败后，线程会**忙等待**，直到它拿到锁，不会释放cpu

   其加锁和释放锁通过cpu提供的CAS函数，不会进行上下文切换，开销小些

   ![1](https://gitee.com/lifutian66/img/raw/master/img/1.png)

   **注意：**

   ​	1.忙循环 可以用while实现，不过最好还是用cpu提供的**PAUSE**指令实现，可以减少循环等待时的耗电量

   ​	2.忙循环不会释放cpu，其时长与锁主代码执行时长成正比

   ​	3.单核cpu忙循环不会释放cpu，所以除非有 **抢占式调度器** 否则无法在单核上使用

3. **读写锁**

   读锁和写锁  也就是说 分使用场景的 （适用于明确区分读写）

   **原理**：读锁是共享锁，支持并发访问共享资源，写锁是独占锁，写锁获取失败，会阻塞直至获取到写锁

   **适用**：读多写少的场景

   **实现**：**【读优先锁】 【写优先锁】【公平读写锁】**

   

   读优先：

   ![](https://mmbiz.qpic.cn/mmbiz_png/J0g14CUwaZflJahXjfiaG4OvTA9DA2UibzGiaX1mvYx5jzfQaYsG9hYbicIzos7M9SkKz0wWMoxBk9RwyguyWwtricA/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

   写优先：

   ![图片](https://mmbiz.qpic.cn/mmbiz_png/J0g14CUwaZflJahXjfiaG4OvTA9DA2UibzskMiariaXsTzJYibmXK6vGf9fWOlJI6oSaB0ibBIp40Gia5V0VsWclRvttw/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

   读优先锁对于读线程并发性更好，长时间会导致 写线程「饥饿」

   写优先锁对于写线程并发性更好，长时间会导致 读线程「饥饿」

   以上都会出现 饥饿现象，其实还有一种是 【**公平读写锁**】 

   **用队列把获取锁的线程排队，不管是写线程还是读线程都按照先进先出的原则加锁即可**

4. **悲观锁**

   前面提到的互斥锁、自旋锁、读写锁，都是属于悲观锁。

   悲观锁做事比较悲观，它认为**多线程同时修改共享资源的概率比较高，于是很容易出现冲突，所以访问共享资源前，先要上锁**。

5. **乐观锁**

   乐观锁做事比较乐观，它假定冲突的概率很低，它的工作方式是：**先修改完共享资源，再验证这段时间内有没有发生冲突，如果没有其他线程在修改资源，那么操作完成，如果发现有其他线程已经修改过这个资源，就放弃本次操作**。

   

   常见的 SVN 和 Git 也是用了乐观锁的思想，先让用户编辑代码，然后提交的时候，通过版本号来判断是否产生了冲突，发生了冲突的地方，需要我们自己修改后，再重新提交。

   

   乐观锁虽然去除了加锁解锁的操作，但是一旦发生冲突，重试的成本非常高，所以**只有在冲突概率非常低，且加锁成本非常高的场景时，才考虑使用乐观锁。**

#### 2.CountDownLatch，CyclicBarrier，Semaphore

1. CountDownLatch
2. CyclicBarrier
3. Semaphore

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





