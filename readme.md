

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

②禁止指令重排序

为了提高性能，编译器和处理器常常会对既定的代码执行顺序进行指令重排序。

怎么做的？

在适当位置插入内存屏障

写：

![图片](https://gitee.com/lifutian66/img/raw/master/img/640)

读：

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/uChmeeX1FpzhiaXUhn9W2XjuqeziaG1ibdvaOPHe2KysUlTCphhnkoaacAho6ZFv3F4vaetoGu4dUQcvPn4wicvGwA/640?wx_fmt=jpeg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)



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

#### 1.io

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

​	

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





