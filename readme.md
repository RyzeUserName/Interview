

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

1. **CountDownLatch**

   别名闭锁，AQS实现

   构造：CountDownLatch(int count)

   使用：

    		1.初始化个数量

   ​		 2.在需要并行执行的代码前加 CountDownLatch.await() 

   ​		 3.执行 CountDownLatch.countDown() 减少次数，当次数减少到0，await 之后的代码并行执行

   举例：

   ```java
   public class CountDownLatchTest {
       private static final CountDownLatch COUNT_DOWN_LATCH = new CountDownLatch(8);
       public static void main(String[] args) {
           for (int i = 0; i < 8; i++) {
               new ThreadMy(i, COUNT_DOWN_LATCH).start();
               COUNT_DOWN_LATCH.countDown();
                System.out.println("闭锁 剩余 " + COUNT_DOWN_LATCH.getCount());
           }
       }
       static class ThreadMy extends Thread {
           private final int i;
           private final CountDownLatch countDownLatch;
           public ThreadMy(int i, CountDownLatch countDownLatch) {
               this.i = i;
               this.countDownLatch = countDownLatch;
           }
           @Override
           public void run() {
               try {
                   countDownLatch.await();
                   System.out.println("需要并行执行的代码----" + i);
               } catch (Exception e) {
                   System.out.println("发生异常");
               }
           }
       }
   }
   ```

   结果：

   ```txt
   闭锁 剩余 7
   闭锁 剩余 6
   闭锁 剩余 5
   闭锁 剩余 4
   闭锁 剩余 3
   闭锁 剩余 2
   闭锁 剩余 1
   闭锁 剩余 0
   需要并行执行的代码----2
   需要并行执行的代码----3
   需要并行执行的代码----6
   需要并行执行的代码----7
   需要并行执行的代码----0
   需要并行执行的代码----1
   需要并行执行的代码----4
   需要并行执行的代码----5
   ```

   

2. **CyclicBarrier**

   别名： 同步屏障,一组线程到达某个指定 终点 时，相互等待，直到指定数量的线程都达到，才会一起往下走

   实现： ReentrantLock 重入锁

   构造：  CyclicBarrier(int parties, Runnable barrierAction) 

   使用：

   ​	1.初始化 数量 和 需要达到指定位置  执行指定代码

   ​	2.前置准备好了，调用CyclicBarrier.await()  , CyclicBarrier 数量增加1 直到指定数量，各线程并行执行 CyclicBarrier.await()之后的代码

   示例：

   ```java
   public class CyclicBarrierTest {
       public static final Runnable RUNNABLE = () -> {
           System.out.println("都到达，执行需要代码");
       };
       public static final CyclicBarrier CYCLIC_BARRIER = new CyclicBarrier(2, RUNNABLE);
   
       public static void main(String[] args) {
           for (int i = 0; i < 2; i++) {
               new ThreadMy(i, CYCLIC_BARRIER).start();
           }
       }
       static class ThreadMy extends Thread {
           private final int i;
           private final CyclicBarrier cyclicBarrier;
   
           public ThreadMy(int i, CyclicBarrier cyclicBarrier) {
               this.i = i;
               this.cyclicBarrier = cyclicBarrier;
           }
   
           @Override
           public void run() {
               try {
                   System.out.println("准备好了" + i);
                   int await = cyclicBarrier.await();
                   System.out.println(await+"---执行后面----" + i);
               } catch (Exception e) {
                   System.out.println("发生异常");
               }
           }
       }
   }
   ```

   结果：

   ```txt
   准备好了1
   准备好了0
   都到达，执行需要代码
   0---执行后面----0
   1---执行后面----1
   ```

   

3. **Semaphore**

   别名：信号量，用来控制同一时间，资源可被访问的线程数量，一般可用于流量的控制

   构造：Semaphore(int permits, boolean fair)

   ​	后面的参数即 是否是公平锁，默认非公平锁，实现是AQS

   使用：

   ​	1.SEMAPHORE.acquire()  或者 SEMAPHORE.tryAcquire() 或者 SEMAPHORE.acquire(n)获取锁

   ​	2.执行代码

   ​	3.SEMAPHORE.release() 释放 SEMAPHORE.release(n)  释放多个

   示例：

   ```java
   public class SemaphoreTest {
       public static final Semaphore SEMAPHORE = new Semaphore(2);
   
       public static void main(String[] args) {
           for (int i = 0; i < 10; i++) {
               new Thread(() -> {
                   try {
                       SEMAPHORE.acquire();
                       Thread.sleep(2000);
                       System.out.println("执行");
                       SEMAPHORE.release();
                   } catch (InterruptedException e) {
                       e.printStackTrace();
                   }
               }).start();
           }
       }
   }
   ```

   结果：

   每隔两秒 打印一组 

#### 3.Volatile

​	java 线程与内存的关系

​	![](https://mmbiz.qpic.cn/mmbiz_jpg/uChmeeX1FpzhiaXUhn9W2XjuqeziaG1ibdvOgPyiaPib3U7oR6ZS77CqlAVp7BkTxS30UhDN1X6YJRfCGQadBP6xd9Q/640?wx_fmt=jpeg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

①不同线程之间存在可见性问题

解决：

1.加锁

2.Volatile 修饰共享变量

Volatile 的实现 是依靠 处理器访问缓存遵循一些一致性协议（MSI、MESI、MOSI、Synapse、Firefly及DragonProtocol等）

MESI（M修改 E独享 S共享 I 无效） 嗅探机制

1.变量被①线程 读取 ①线程变量状态E 

2.变量被②线程 读取 ②线程变量状态S ①线程也变成S

3.①线程修改变量 状态变为M ②变为 I

4.①线程变量写回主内存，主内存同步变量给②线程 都变成S

I到 S 会有时间差， 频繁操作导致可见性的结果并不一定准确 



②禁止指令重排序

为了提高性能，编译器和处理器常常会对既定的代码执行顺序进行指令重排序。

怎么做的？

在适当位置插入内存屏障

写：

![图片](https://gitee.com/lifutian66/img/raw/master/img/640)

读：

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/uChmeeX1FpzhiaXUhn9W2XjuqeziaG1ibdvaOPHe2KysUlTCphhnkoaacAho6ZFv3F4vaetoGu4dUQcvPn4wicvGwA/640?wx_fmt=jpeg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

Volatile 只能保证可见性，有序性， 不能保证原子性

#### 4.ThreadLocal

用作数据隔离，保证本线程的数据不会被其他线程篡改

**1.用案例**： 

1.TransactionSynchronizationManager

![image-20210401165349362](https://gitee.com/lifutian66/img/raw/master/img/image-20210401165349362.png)

2.Spring的事务主要是ThreadLocal和AOP去 , 每个线程自己的链接是靠ThreadLocal保存的

3.SimpleDataFormat.parse(), 内部有一个Calendar对象，调用SimpleDataFormat的parse()方法会先调用Calendar.clear（），然后调用Calendar.add(),多线程环境会有问题，可以使用ThreadLocal 减少 SimpleDataFormat 的数量

4.多场景的cookie，session等数据隔离都是通过ThreadLocal

**2.原理**

![](https://gitee.com/lifutian66/img/raw/master/img/Snipaste_2021-04-01_17-21-27.png)



![](https://gitee.com/lifutian66/img/raw/master/img/Snipaste_2021-04-01_17-22-00.png)

看出：get() 方法,从当前线程的threadLocals属性 取出值

set  同理 

threadLocals 属性的类型 是 ThreadLocalMap，里面有   Entry[] table 

![](https://gitee.com/lifutian66/img/raw/master/img/Snipaste_2021-04-01_17-30-11.png)

entry 中的 key 是Threadlocal   value就是存取的值

为什么需要数组呢？ 因为一个线程可以有 多个 Threadlocal  

原理：

​	set 方法中

![](https://gitee.com/lifutian66/img/raw/master/img/Snipaste_2021-04-01_18-10-54.png)

 	共享线程的ThreadLocal数据怎么办？

​	使用 InheritableThreadLocal类，父子线程值传递

​	原理：

​	在Thread的构造函数中，我们可以看到 其实是调用了 init() 方法，此方法中，如下图

​	![image-20210402093012432](https://gitee.com/lifutian66/img/raw/master/img/image-20210402093012432.png)

如果父线程中的 inheritableThreadLocals不为空，并且inheritThreadLocals 标识为 true，会将父线程的inheritableThreadLocals 往下传递



**问题：**

ThreadLocalMap 中的存值的 entry key 是Threadlocal  弱引用的，会导致 内存泄露

即：ThreadLocal在没有外部强引用时 GC时被回收，如果创建ThreadLocal的线程一直持续运行，那么这个Entry对象中的value就有可能一直得不到回收，

发生内存泄露（线程池）

 **解决:**

使用完 ThreadLocal remove 其实在源码 （get set 都会处理被回收的引用）

 **为什么ThreadLocalMap的key要设计成弱引用？**

![1](https://gitee.com/lifutian66/img/raw/master/img/1.jpg)

 当前线程的属性ThreadLocalMap key弱引用，是为了 当 Threadlocal不被强引用的时 发生gc 回收即可，否则 key强引用，一直会被ThreadLocalMap 强引用

 导致ThreadLocal  不会被回收，导致key也发生内存泄漏 



#### 5.Synchronized

**有序性、可见性、原子性**

1.有序性  as-if-serial

2.可见性  缓存一致性 协议

3.原子性 确保同一时间只有一个线程能拿到锁

 **底层实现**

代码：

```java
public class Synchronized {
    public synchronized void husband() {
        synchronized (new Synchronized()) {

        }
    }
}
```

在其编译的.class 文件执行

```txt
javap -p -v -c  Synchronized.class
```

看到执行的结果：

```
Classfile /D:/study/java8/target/classes/com/example/java8/inter/Synchronized.class
  Last modified 2021-4-2; size 499 bytes
  MD5 checksum b6a4702ac06de7cf1f0f1cf2039e8b52
  Compiled from "Synchronized.java"
public class com.example.java8.inter.Synchronized
  minor version: 0
  major version: 52
  flags: ACC_PUBLIC, ACC_SUPER
Constant pool:
   #1 = Methodref          #4.#19         // java/lang/Object."<init>":()V
   #2 = Class              #20            // com/example/java8/inter/Synchronized
   #3 = Methodref          #2.#19         // com/example/java8/inter/Synchronized."<init>":()V
   #4 = Class              #21            // java/lang/Object
   #5 = Utf8               <init>
   #6 = Utf8               ()V
   #7 = Utf8               Code
   #8 = Utf8               LineNumberTable
   #9 = Utf8               LocalVariableTable
  #10 = Utf8               this
  #11 = Utf8               Lcom/example/java8/inter/Synchronized;
  #12 = Utf8               husband
  #13 = Utf8               StackMapTable
  #14 = Class              #20            // com/example/java8/inter/Synchronized
  #15 = Class              #21            // java/lang/Object
  #16 = Class              #22            // java/lang/Throwable
  #17 = Utf8               SourceFile
  #18 = Utf8               Synchronized.java
  #19 = NameAndType        #5:#6          // "<init>":()V
  #20 = Utf8               com/example/java8/inter/Synchronized
  #21 = Utf8               java/lang/Object
  #22 = Utf8               java/lang/Throwable
{
  public com.example.java8.inter.Synchronized();
    descriptor: ()V
    flags: ACC_PUBLIC
    Code:
      stack=1, locals=1, args_size=1
         0: aload_0
         1: invokespecial #1                  // Method java/lang/Object."<init>":()V
         4: return
      LineNumberTable:
        line 7: 0
      LocalVariableTable:
        Start  Length  Slot  Name   Signature
            0       5     0  this   Lcom/example/java8/inter/Synchronized;

  public synchronized void husband(); 
    descriptor: ()V
    flags: ACC_PUBLIC, ACC_SYNCHRONIZED // 这里看到同步方法 标识 是 ACC_SYNCHRONIZED
    Code:
      stack=2, locals=3, args_size=1
         0: new           #2                  // class com/example/java8/inter/Synchronized
         3: dup
         4: invokespecial #3                  // Method "<init>":()V
         7: dup
         8: astore_1
         9: monitorenter					//同步代码进入
        10: aload_1
        11: monitorexit  					//同步代码出来
        12: goto          20
        15: astore_2
        16: aload_1
        17: monitorexit						//同步代码出来
        18: aload_2
        19: athrow
        20: return
      Exception table:
         from    to  target type
            10    12    15   any
            15    18    15   any
      LineNumberTable:
        line 9: 0
        line 11: 10
        line 12: 20
      LocalVariableTable:
        Start  Length  Slot  Name   Signature
            0      21     0  this   Lcom/example/java8/inter/Synchronized;
      StackMapTable: number_of_entries = 2
        frame_type = 255 /* full_frame */
          offset_delta = 15
          locals = [ class com/example/java8/inter/Synchronized, class java/lang/Object ]
          stack = [ class java/lang/Throwable ]
        frame_type = 250 /* chop */
          offset_delta = 4
}
SourceFile: "Synchronized.java"

```

同步方法标识 **ACC_SYNCHRONIZED** 之后隐式调动 monitorenter和monitorexit  指令

同步代码 就是 monitorenter 进入 monitorexit  退出



1.6之前是重量级锁，是ObjectMonitor调用的过程，以及Linux内核的复杂运行机制决定的，大量的系统资源消耗，所以效率才低

之后对锁做了优化

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/uChmeeX1FpyB6WkMTL2IUapfTtGH6FFOiaEtKGL0EickicibDjwEqKRK8rMUz7kxlSmiapXejO8fmmcyBGBSSju8TYQ/640?wx_fmt=jpeg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

1.偏向锁

**偏向锁的目标是，减少无竞争且只有一个线程使用锁的情况下，使用轻量级锁产生的性能消耗**

在Mark Word中CAS记录owner,偏向锁获取成功，记录锁状态为偏向锁,本线程可以不用cas也可以获取锁

否则，说明有其他线程竞争，膨胀为轻量级锁

采用cas 方式

2.轻量级锁

**轻量级锁的目标是，减少无实际竞争情况下，使用重量级锁产生的性能消耗**

仅仅将Mark Word中的部分字节CAS更新指向线程栈中的Lock Record，如果更新成功，则轻量级锁获取成功，

记录锁状态为轻量级锁 ，否则，说明已经有线程获得了轻量级锁，目前发生了锁竞争（不适合继续使用轻量级锁）,接下来膨胀为重量级锁
采用cas 方式

3.自旋锁

**自旋锁的目标是，通过自旋，减少线程上线文切换**

#### 6.AQS

AbstractQueuedSynchronizer

**抽象 队列  同步器**

ReentrantLock、ReentrantReadWriteLock、CountDownLatch、Semaphore等都是基于AQS 实现

如图：

![Snipaste_2021-04-06_14-01-29](https://gitee.com/lifutian66/img/raw/master/img/Snipaste_2021-04-06_14-01-29.png)

ReentrantLock、ReentrantReadWriteLock、Semaphore 均有公平锁和非公平锁实现

CountDownLatch 只有非公平锁实现

![](https://gitee.com/lifutian66/img/raw/master/img/Snipaste_2021-04-06_14-12-13.png)

其实公平锁与非公平锁 区别 ：是否根据请求共享资源先后顺序为依据，获取共享资源

**设计模式**是**模板模式** 

##### 1.**核心数据结构**：

**双向链表 + state(锁状态)**

![222324](https://gitee.com/lifutian66/img/raw/master/img/222324.png)

其每个线程都会封装成Node

具体实现：

 ![Snipaste_2021-04-06_18-22-33](https://gitee.com/lifutian66/img/raw/master/img/Snipaste_2021-04-06_18-22-33.png)

**排他锁(exclusive)**  只有一个线程可以访问共享变量    **重入锁**

**共享锁(shared)**  允许多个线程同时访问   **信号量**

关于 **waitStatus** 

CANCELLED：表示线程取消    

SIGNAL：表示后续节点需要被唤醒   

CONDITION：线程等待在条件变量队列中

PROPAGATE:  在共享模式下，无条件传播releaseShared状态

**重入锁  当有多条件时，内部维护 多条件等待队列**

![](https://gitee.com/lifutian66/img/raw/master/img/saada.png)

条件等待队列数据结构就是  **ConditionObject**，其内部依然是 **Node** ， firstWaiter，lastWaiter 组成链表

**底层操作：CAS**

##### **2.方法**：

###### 1.获取锁（许可）

1. 独占锁

**acquire**

![](https://gitee.com/lifutian66/img/raw/master/img/Snipaste_2021-04-07_10-55-51.png)

tryAcquire()函数是个未实现的方法，需要自己实现

addWaiter 添加到队列

![](https://gitee.com/lifutian66/img/raw/master/img/Snipaste_2021-04-07_11-03-00.png)

addWaiter 返回 node 对象

acquireQueued 方法

![](https://gitee.com/lifutian66/img/raw/master/img/Snipaste_2021-04-07_11-18-28.png)

打断返回 true，执行   selfInterrupt方法  就是  Thread.currentThread().interrupt() 

**acquireInterruptibly**

![image-20210407112347132](https://gitee.com/lifutian66/img/raw/master/img/image-20210407112347132.png)

**tryAcquireNanos**  增加了最长等待 时间

![](https://gitee.com/lifutian66/img/raw/master/img/Snipaste_2021-04-07_13-46-09.png)

2. 共享锁

**acquireShared**

![](https://gitee.com/lifutian66/img/raw/master/img/Snipaste_2021-04-07_14-06-29.png)

tryAcquireShared 未实现，模板方法等待实现

**acquireSharedInterruptibly**

![image-20210407140840145](https://gitee.com/lifutian66/img/raw/master/img/image-20210407140840145.png)

增加了线程是否被打断的判断，之后的代码类似上面的**acquireShared** 

**tryAcquireSharedNanos** 增加了最长等待时间

![](https://gitee.com/lifutian66/img/raw/master/img/Snipaste_2021-04-07_14-10-44.png)

在 **acquireShared** 方法基础上增加了 最长等待时间 判断

###### 2.释放锁（许可）

独占锁 

release

![](https://gitee.com/lifutian66/img/raw/master/img/Snipaste_2021-04-08_16-45-04.png)

tryRelease 方法是个模板方法 待实现，unparkSuccessor 方法 唤醒下一个线程

![](https://gitee.com/lifutian66/img/raw/master/img/Snipaste_2021-04-07_14-24-17.png)

先查看下一个节点 是不是可以激活的，如果不是，查找下个可唤醒线程是从尾 到 头 查找

原因：

```java
private Node enq(final Node node) {
        for (;;) {
            Node t = tail;
            if (t == null) { // Must initialize
                if (compareAndSetHead(new Node()))
                    tail = head;
            } else {
                //看这里
                node.prev = t;
                if (compareAndSetTail(t, node)) {
                    t.next = node;
                    return t;
                }
            }
        }
}
```

新节点pre指向tail，tail指向新节点，这里后继指向前驱的指针是由CAS操作保证线程安全的。

而cas操作之后t.next=node之前，可能会有其他线程进来。所以出现了问题，从尾部向前遍历是一定能遍历到所有的节点



releaseShared

![](https://gitee.com/lifutian66/img/raw/master/img/Snipaste_2021-04-08_17-15-02.png)

tryAcquireShared 是个模板方法 待实现

###### 3.ReentrantLock

有公平锁和非公平锁

![](https://gitee.com/lifutian66/img/raw/master/img/Snipaste_2021-04-09_17-38-43.png)

​	其区别就是 hasQueuedPredecessors（） 这个方法其实就是查看 是否有等待更久的线程 

​	其Condition  源自 AbstractQueuedSynchronizer  ConditionObject

​	ConditionObject# await  其实就是将其添加到等待队列    

​	ConditionObject# signal  唤醒 firstWaiter  不是null 的

​	ConditionObject# signalAll  重复 signal   全部唤醒

###### 4.ReentrantReadWriteLock

公平锁和非公平锁 区别 writerShouldBlock /readerShouldBlock 返回  hasQueuedPredecessors/false	

ReadLock/WriteLock 分别调用  acquireShared/acquire 获取锁  释放锁同理

###### 5.CountDownLatch

 其实现了  AbstractQueuedSynchronizer 的  tryAcquireShared /tryReleaseShared 

###### 6.Semaphore

公平锁和非公平锁 区别 tryAcquire/tryAcquireShared 获取锁

### 3.tcp/http

计算机基础：

![img](https://gitee.com/lifutian66/img/raw/master/img/091053xho6f2omicyomohn.jpg)

计算机组成：CPU，内存，网络接口等

网卡接收数据的过程： 

​	1.网卡接收到网线传来的数据

​	2.**网卡向cpu发出一个中断信号，操作系统便能得知有新数据到来**，再通过网卡**中断程序**去处理数据

​	3.网络数据写入到对应socket的接收缓冲区里面

​	4.唤醒对应的线程

操作系统如何知道网络数据对应于哪个socket？

socket对应着端口号，而网络数据包中包含了ip和端口的信息

如何同时监视多个socket的数据？

**多路复用**，也就是 select poll  epoll



#### 1.io

io：输入输出

![img](https://gitee.com/lifutian66/img/raw/master/img/215778a93e00c1e250154d3acd83309c.webp)



**硬 件 层（Hardware）**：包括和我们熟知的和IO相关的CPU、内存、磁盘和网卡几个硬件；

**内核空间（Kernel Space）** ：计算机一部分核心软件独立于普通应用程序，运行在较高的特权级别上，它们驻留在被保护的内存空间上，拥有访问硬件设备的所有权限，Linux将此称为内核空间

**用户空间 （**User Space**）**：用户空间中的代码运行在较低的特权级别上，只能看到允许它们使用的部分系统资源，并且不能使用某些特定的系统功能，也不能直接访问内核空间和硬件设备，以及其他一些具体的使用限制。

**用户操作底层？**

操作系统在内核开辟了一块唯一且合法的**系统调用** System Call Interface( 可供用户调用的底层硬件的API)，不用进程就可以调用api，内核在调用底层硬件，用户态到内核态切换     比如：内存映射mmap()、文件操作类的open()、IO读写read()、write()等等。

1.BIO

阻塞IO，io操作会阻塞 

缺点：**阻塞** 

优点:**阻塞是不会消耗CPU资源** 

2.BIO + 线程池

多线程处理  

缺点：依赖线程，线程占用资源，线程上线文切换成本较高    

优点：处理少量并发

3.NIO非阻塞模型

通过单线程或者少量线程达到处理大量客户端请求的目的，“非阻塞”可以理解成系统调用API级别的，而真正底层的IO操作都是阻塞的

Accept官方文档对其非阻塞部分的描述，"**flags**"参数设成"**SOCK_NONBLOCK**"就可以达到非阻塞的目的，非阻塞之后线程会一直处理轮询调用，这时候可以通过每次返回特殊的异常码“**EAGAIN**”或"**EWOULDBLOCK**"告诉主程序还没有连接到达可以继续轮询

大致意思：**用户进程需要不断去主动询问内核数据准备好了没有！**

优点：api级非阻塞编程

缺点：用户进程不断切换到内核态，对连接状态或读写数据做轮询

4.IO多路复用模型

频繁的调用系统函数，一次性把数据传递过去就省去了频繁的系统间调用，只不过入参这个“集合”需要你注册/填写感兴趣的事件，读fd、写fd或者连接状态的fd等，然后交给内核帮你进行处理。几个系统调用 **- select()、poll()、epoll()**

**select()**

```
/**
 ``* select()系统调用
 ``*
 ``* 参数列表：
 ``*   nfds    - 值为最大的文件描述符+1
 ``*  *readfds  - 用户检查可读性
 ``*  *writefds  - 用户检查可写性
 ``*  *exceptfds - 用于检查外带数据
 ``*  *timeout  - 超时时间的结构体指针
 ``*/
int` `select（``int` `nfds, fd_set *readfds, fd_set *writefds, fd_set *exceptfds, struct timeval *timeout）;
```

内核使用**select()**为用户进程提供了类似批量的接口，函数本身也会一直阻塞直到有fd为就绪状态返回

readfds 是全部读事件的集合，writefds是全部写事件的集合

![img](https://gitee.com/lifutian66/img/raw/master/img/42b1483530de04358157e1f2083d7862.webp)

缺点：

- 复杂度O(n)，轮询的任务交给了内核来做，复杂度并没有变化，数据取出后也需要轮询哪个fd上发生了变动；
- 用户态还是需要不断切换到内核态，直到所有的fds数据读取结束，整体开销依然很大；
- fd_set有大小的限制，目前被硬编码成了**1024**；
- fd_set不可重用，每次操作完都必须重置；

**poll()**

```
/**
 ``* poll()系统调用
 ``*
 ``* 参数列表：
 ``*  *fds     - pollfd结构体
 ``*   nfds    - 要监视的描述符的数量
 ``*   timeout   - 等待时间
 ``*/
int` `poll（struct pollfd *fds, nfds_t nfds, ``int` `*timeout）;
 
 
### pollfd的结构体
struct pollfd{
　``int` `fd；``// 文件描述符
　``short` `event；``// 请求的事件
　``short` `revent；``// 返回的事件
}
```

**poll()**和**select()**是非常相似的，唯一的区别在于**poll()**摒弃掉了位图算法，使用自定义的pollfd 结果体，并通过event变量注册感兴趣的可读可写事件（**POLLIN、POLLOUT**），最后把 **pollfd** 交给内核。当有读写事件触发的时候，我们可以通过轮询 **pollfd**，判断revent确定该fd是否发生了可读可写事件。

![](https://gitee.com/lifutian66/img/raw/master/img/289b491c76209266080a21a2d47e8705.webp)

优点：没有了位图算法的1024大小限制

缺点：依然解决不了用户态切换内核态切换以及O(n)复杂度问题

**epoll()**  

```
/**
 * 返回专用的文件描述符
 */
int epoll_create（int size）;
/**
 ``* epoll_ctl()系统调用
 ``*
 ``* 参数列表：
 ``*   epfd    - 由epoll_create()返回的epoll专用的文件描述符
 ``*   op     - 要进行的操作例如注册事件,可能的取值:注册-EPOLL_CTL_ADD、修改-EPOLL_CTL_MOD、删除-EPOLL_CTL_DEL
 ``*   fd     - 关联的文件描述符
 ``*   event   - 指向epoll_event的指针
 ``*/
int` `epoll_ctl（``int` `epfd, ``int` `op, ``int` `fd , struce epoll_event *event ）;
/**
 * epoll_wait()返回n个可读可写的fds
 *
 * 参数列表：
 *     epfd           - 由epoll_create()返回的epoll专用的文件描述符
 *     epoll_event    - 要进行的操作例如注册事件,可能的取值:注册-EPOLL_CTL_ADD、修改-EPOLL_CTL_MOD、删除-EPOLL_CTL_DEL
 *     maxevents      - 每次能处理的事件数
 *     timeout        - 等待I/O事件发生的超时值；-1相当于阻塞，0相当于非阻塞。一般用-1即可
 */
int epoll_wait（int epfd, struce epoll_event *event , int maxevents, int timeout）;
```

Nginx、Redis都广泛地使用了此种模式, 将维护等待队列 和 阻塞进程 步骤分开，效率就能得到提升。

epoll 采用三步走

​	1.epoll_create  用户进程通过 通过该函数在内核空间里面创建了一块空间，并返回描述此空间的fd

​	2.epoll_ctl 通过自定义 epoll_event结构体在fd 注册感兴趣的事件

​	3.epoll_wait 一直阻塞等待，直到硬盘、网卡等硬件设备数据准备完成后发起**硬中断**，中断CPU，CPU会立即执行数据拷贝工作，数据从磁盘缓冲传输到内核缓

​		冲，同时将准备完成的fd放到就绪队列中供用户态进行读取。用户态阻塞停止，接收到**具体数量**的可读写的fds，返回用户态进行数据处理

整体：

![](https://gitee.com/lifutian66/img/raw/master/img/af4673f2b7b255319bfdf11b57391d7d.webp)

优点：没有频繁的用户态到内核态的切换，O(1)复杂度，返回的"nfds"是一个确定的可读写的数量

缺点：独立创建单独空间









流：代表任何有能力产出数据的数据源对象或者是有能力接受数据的接收端对象

同步阻塞io： 没资源时挂起 不能马上返回，等待资源可用

![](https://gitee.com/lifutian66/img/raw/master/img/sksk.png)

阻塞等待浪费资源，不适合并发

解决：**1.IO复用的模型**

​				多个连接共享一个阻塞对象,应用程序只会在一个阻塞对象上等待,当某个连接有新的数据处理，操作系统直接**通知**应用程序，

​				线程从阻塞状态返回并开始业务处理

​			**2.线程池复用的方式**

​				将连接完成后的业务处理任务分配给线程，一个线程处理多个连接的业务。IO复用结合线程池的方案即Reactor模式。

​				基于事件驱动的 处理模型，将处理变成对应的事件

​								![](https://gitee.com/lifutian66/img/raw/master/img/1111111.png)

​			通常网络处理：

![11111112](https://gitee.com/lifutian66/img/raw/master/img/11111112.png)

#### 2.tcp/ip

TCP 即 Transmission Control Protocol，可以看到是一个传输控制协议，重点就在这个**控制**。

控制可靠、按序地传输以及端与端之间的流量控制

**tcp面向连接**，**所谓的连接其实只是双方都维护了一个状态，通过每一次通信来维护状态的变更**，使得看起来好像有一条线关联了对方。

tcp协议头 有：

Seq 就是 Sequence Number 即序号，它是用来解决乱序问题的。

ACK 就是 Acknowledgement Numer 即确认号，它是用来解决丢包情况的，告诉发送方这个包我收到啦

##### **1.三次握手**

​		握手就是为了初始化双方seq序号

​		![](https://gitee.com/lifutian66/img/raw/master/img/12.png)

 SYN 全称  Synchronize Sequence Numbers，这个序号是用来保证之后传输数据的顺序性

 SYN +ACK 保证了 顺序和数据丢失

 SYN超时之后 阶梯性重试 1s  2s 4s 8s 16s 32s 之后便断开连接（SYN攻击 ：减少重试次数 、开启tcp_syncookies、减少重试的次数、增加 SYN 队列数 ）

**握手的重点就是同步初始序列号**，这种情况也完成了同步的目标

##### **2.四次挥手**

为什么挥手需要四次？**因为 TCP 是全双工协议**，也就是说双方都要关闭，每一方都向对方发送 FIN 和回应 ACK。

![](https://gitee.com/lifutian66/img/raw/master/img/13.png)

​	TIME_WAIT 是会等待 2MSL （Maximum Segment Lifetime 即报文最长生存时间）

​		原因：1.怕被动关闭方没有收到最后的 ACK

​					2.可能会重连 需要时间处理残留数据 

​		产生问题：

​				 资源占用 端口占用

​		解决：

​				**服务端不要主动关闭，把主动关闭方放到客户端**

​	**超时重试**：事件驱动，网络不稳定的如果传输的包对方没收到, 那么就必须重传，回复ack **只能回复确认最大连续收到包**，发送方需要等待

​						那么等待多长时间？ 基于来回时间  统计计算出 

​	**快速重传**：数据驱动的重传, 如果网络状况好的时候，只是恰巧丢包了，那等这么长时间就没必要(连续收到三次相同 ACK 证明当前网络状况是 ok 的)

​	**滑动窗口**:   控制传送速率

​	**拥塞控制**：网络差，没收到ack 无脑重传 只会加剧 网络 阻塞

​	1、慢启动，探探路。2、拥塞避免，感觉差不多了减速看看 3、拥塞发生快速重传/恢复

分层：

![](https://gitee.com/lifutian66/img/raw/master/img/15.png)

#### 3.linux 命令

#### 4.http

​	HyperText Transfer Protocol   超文本传输协议

​	**HTTP的特点**： 无状态 、无连接

​	**HTTP/1.0版的主要缺点**：每个TCP连接只能发送一个请求，发送数据完毕后，连接就关闭了，如果还要请求就必须要新建一个请求连接

​	HTTP1.1虽然是无状态协议，但是为了实现期望的保持状态功能，于是引入了Cookie技术，有了Cookie，和HTTP协议通信，就可以管理状态了。

​	HTTP或者HTTPS协议请求的资源由 统一资源标识符（Uniform Resource Identifiers，URI）来标识

​	**HTTP状态码**（HTTP Status Code）是用以表示网页服务器HTTP响应状态的3位数字代码，**HTTP状态码是服务器端返回给客户端的**

​	200 成功	

​	202 服务器已经接受请求，但尚未处理

​	204 服务器成功处理了请求，但不需要返回如何实体内容。

​	301 永久性的重定向

​	302 临时跳转

​	304 被请求的资源内容没有发生更改

​	400 客户端请求的语法错误，服务器无法理解

​	401 请求要求用户的身份认证

​	403 为服务器已经接收请求，但是被拒绝执行

​	404 所请求的资源无法找到 

​	500 为服务器内部错误，无法处理请求

​	502 作为网关或者代理工作的服务器尝试执行请求时，从远程服务器接收到了一个无效的响应

#### 5.https

​	HTTP 有着一个致命的缺陷，内容是**明文传输**

​	HyperText Transfer Protocol Secure  超文本传输安全协议，数据通信仍然是HTTP，但利用**SSL/TLS加密数据包**。

​	**过程：**

1. 用户在浏览器发起HTTPS请求，默认使用服务端的443端口进行连接

 	2. 服务端在使用HTTPS前，去经过认证的CA机构申请颁发一份**数字证书**，数字证书里包含有证书持有者、证书有效期、公钥等信息，证书内会附带一个**公钥Pub**，而与之对应的**私钥Private**保留在服务端不公开；
 	3. 服务端收到请求，返回配置好的包含**公钥Pub**的证书给客户端；
 	4. 客户端收到**证书**，校验合法性，主要包括是否在有效期内、证书的域名与请求的域名是否匹配，上一级证书是否有效（递归判断，直到判断到系统内置或浏览器配置好的根证书），如果不通过，则显示HTTPS警告信息，如果通过则继续；
 	5. 客户端生成一个用于对称加密的**随机Key**，并用证书内的**公钥Pub**进行加密，发送给服务端；
 	6. 服务端收到**随机Key**的密文，使用与**公钥Pub**配对的**私钥Private**进行解密，得到客户端真正想发送的**随机Key**；
 	7. 服务端使用客户端发送过来的**随机Key**对要传输的HTTP数据进行对称加密，将密文返回客户端；
 	8. 客户端使用**随机Key**对称解密密文，得到HTTP数据明文；
 	9. 后续HTTPS请求使用之前交换好的**随机Key**进行对称加解密。

#### 6.netty

​	同步和异步关注的是**消息通信机制**   同步不返回就等待，异步在发出调用之后，这个调用就直接返回了，没有返回结果

​	阻塞和非阻塞关注的是**程序在等待调用结果时的状态.**  阻塞当前线程会被挂起，非阻塞不会阻塞当前线程

​	**BIO**是一个同步并阻塞的IO模式，**传统的  java.io 包**，它基于流模型实现，提供了我们最熟知的一些 IO 功能

​	面向流的，一位置每次从流中读取字节，直至读取完全部字节，他们没有缓存在任何地方，因此是不能前后移动流中数据，需要移动或者操作的话需要将其缓存	到缓冲区。

​	**NIO** 是一种同步非阻塞的 I/O 模型 ，支持面向缓冲的，基于通道的 I/O 操作方法

​	面向缓冲区的，数据读取到一个稍后处理的缓冲区，当然可以前后移动或者操作缓冲区数据。

##### 1.Netty 为什么采用NIO？

​	1.Netty不看重Windows上的使用，在Linux系统上，AIO的底层实现仍使用EPOLL，没有很好实现AIO，因此在性能上没有明显的优势，而且被JDK封装了一层不		容易深度优化

​	2.Netty整体架构是reactor模型, 而AIO是proactor模型, 混合在一起会非常混乱,把AIO也改造成reactor模型看起来是把epoll绕个弯又绕回来

​	3.AIO还有个缺点是接收数据需要预先分配缓存, 而不是NIO那种需要接收时才需要分配缓存, 所以对连接数量非常大但流量小的情况, 内存浪费很多

​	4.Linux上AIO不够成熟，处理回调结果速度跟不到处理需求，比如外卖员太少，顾客太多，供不应求，造成处理速度有瓶颈（待验证）

##### 2.代码

​	Server 代码

```java
 public static void main(String[] args) throws InterruptedException {
        NioEventLoopGroup parent = new NioEventLoopGroup();
        NioEventLoopGroup children = new NioEventLoopGroup();
        try {
            ServerBootstrap serverBootstrap = new ServerBootstrap()
                .group(parent, children)
                .channel(NioServerSocketChannel.class)
                .childHandler(new ChannelInitializer<SocketChannel>() {
                    @Override
                    protected void initChannel(SocketChannel ch) throws Exception {
                        ChannelPipeline pipeline = ch.pipeline();
                        pipeline.addLast(new StringDecoder());
                        pipeline.addLast(new StringEncoder());
                        pipeline.addLast(new SomeSocketServerHandler());

                    }
                });
            ChannelFuture future = serverBootstrap.bind(9999).sync();
            System.out.println("服务器已启动。。。");
            future.channel().closeFuture().sync();
        } catch (Exception e) {
            e.printStackTrace();
        } finally {
            parent.shutdownGracefully();
            children.shutdownGracefully();
        }
    }

    private static class SomeSocketServerHandler extends ChannelInboundHandlerAdapter {
        @Override
        public void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception {
            System.out.println("客户端地址 ====== " + ctx.channel().remoteAddress());
            System.out.println("收到客户端 ====== " + msg);
            ctx.channel().writeAndFlush("服务端回复：" + UUID.randomUUID());
            ctx.fireChannelActive();
            TimeUnit.MILLISECONDS.sleep(500);
        }
    }
```

Client 代码：

```java
 public static void main(String[] args) throws InterruptedException {
        NioEventLoopGroup work = new NioEventLoopGroup();
        try {
            Bootstrap bootstrap = new Bootstrap()
                .group(work)
                .channel(NioSocketChannel.class)
                .handler(new ChannelInitializer<SocketChannel>() {
                    @Override
                    protected void initChannel(SocketChannel ch) throws Exception {
                        ChannelPipeline pipeline = ch.pipeline();
                        pipeline.addLast(new StringDecoder());
                        pipeline.addLast(new StringEncoder());
                        pipeline.addLast(new SomeSocketServerHandler());

                    }
                });
            ChannelFuture future = bootstrap.connect("localhost", 9999).sync();
            System.out.println("客户端已启动。。。");
            future.channel().closeFuture().sync();
        } catch (Exception e) {
            e.printStackTrace();
        } finally {
            if (work != null) {
                work.shutdownGracefully();
            }
        }
    }

    private static class SomeSocketServerHandler extends ChannelInboundHandlerAdapter {
        @Override
        public void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception {
            System.out.println("收到了" + msg);
            ctx.channel().writeAndFlush("客户端 写出 : " + System.currentTimeMillis());
            TimeUnit.MILLISECONDS.sleep(500);
        }

        @Override
        public void channelActive(ChannelHandlerContext ctx) {
            ctx.channel().writeAndFlush("客户端开始 对话");
        }
    }
```

##### 3.**Netty 核心组件**

​	Channel   类似 连接socket 

​	EventLoop 相当于线程池  一个Channel 绑定一个  但是一个EventLoop 可能会分配给一个或多个Channel

​	ChannelHandler    事件就是 网络事件的出入站、用户自定义的事件等，而ChannelHandler 则是对应具体事件的处理

​	ChannelPipeline 管道 与channel 永久性的分配 1:1 

​	ChannelFuture  异步操作结果的占位符，它在未来的某个时刻完成，并提供对其结果的访问 基于ChannelFutureListener 即监听器

​	ByteBuf  堆缓冲区  （ 堆缓冲区 、直接缓冲区 、复合缓冲区）

##### 4.详细介绍

###### 1.EventLoop **NioEventLoopGroup** 

代码继承如图：

![](https://gitee.com/lifutian66/img/raw/master/img/Snipaste_2021-04-15_17-21-44.png)

EventLoop 继承自SingleThreadEventLoo 间接继承 ExecutorService 是 单线程的 线程池

NioEventLoopGroup 继承自 MultithreadEventLoopGroup 间接继承 ExecutorService   是 多线程的 线程池

Executor  必然拥有一个 `execute(Runnable command)` 的实现方法

NioEventLoop 的 `execute()` 实现方法在其父类  SingleThreadEventExecutor 中

![](https://gitee.com/lifutian66/img/raw/master/img/Snipaste_2021-04-15_17-36-12.png)

其内部调用 `startThread()`方法，启动线程

![](https://gitee.com/lifutian66/img/raw/master/img/Snipaste_2021-04-15_17-36-32.png)

![](https://gitee.com/lifutian66/img/raw/master/img/Snipaste_2021-04-15_17-37-37.png)

最终调用的是 SingleThreadEventLoo  内部实例属性 Executor 的 `execute（）`方法

也就是说：

1. NioEventLoop 本身就是一个 Executor。
2. NioEventLoop 内部封装这一个新的线程 Executor 成员。
3. NioEventLoop 有两个 `execute` 方法，除了本身的 `execute()` 方法对应的还有成员属性 Executor  对应的 `execute()` 方法。

而  EventExecutorGroup的 `execute()` 实现方法在其父类的父类   AbstractEventExecutorGroup 中

![](https://gitee.com/lifutian66/img/raw/master/img/Snipaste_2021-04-15_17-47-30.png)

最终调用的是 内部实例属性 Executor 的 `execute（）`方法

也就是说：

1. NioEventLoopGroup 是一个线程池线程 Executor。

2. NioEventLoopGroup 也封装了一个线程 Executor。

3. NioEventLoopGroup 也有两个 `execute()`方法。

**NioEventLoopGroup 初始化**

最终调用

   ![](https://gitee.com/lifutian66/img/raw/master/img/Snipaste_2021-04-15_18-27-051.png)

数组 children （类型 NioEventLoop） 循环调用 `newChild()`方法 初始化

![](https://gitee.com/lifutian66/img/raw/master/img/Snipaste_2021-04-16_09-47-13.png)

具体代码：

![](https://gitee.com/lifutian66/img/raw/master/img/Snipaste_2021-04-16_17-27-38.png)

 也就是 NioEvnetLoopGroup  其实就是个线程池，内部NioEventLoop 的数组，每个线程执行时 调用 总executor 最终完成线程调用

###### 2.Bootstrap  ServerBootstrap 

继承体系

![image-20210420151304429](https://gitee.com/lifutian66/img/raw/master/img/image-20210420151304429.png)

ServerBootstrap 服务器的启动配置类

Bootstrap 客户端的启动配置类

###### 3.

4.

Reactor模式：主动，能收了你跟俺说一声。

使用同步IO，即业务线程处理数据需要主动等待或询问，主要特点是利用epoll监听listen描述符是否有相应，及时将客户连接信息放于一个队列，epoll和队列都是在主进程/线程中，由子进程/线程来接管各个描述符，对描述符进行下一步操作，包括connect和数据读写。主程读写就绪事件

![image](https://gitee.com/lifutian66/img/raw/master/img/943117-20160512230858562-1336019094.png)

Preactor模式：被动，你给我收十个字节，收好了跟俺说一声。

Preactor模式完全将IO处理和业务分离，使用异步IO模型，即内核完成数据处理后主动通知给应用处理，主进程/线程不仅要完成listen任务，还需要完成内核数据缓冲区的映射，直接将数据buff传递给业务线程，业务线程只需要处理业务逻辑即可。



![image](https://gitee.com/lifutian66/img/raw/master/img/943117-20160512230902249-663518390.png)



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





