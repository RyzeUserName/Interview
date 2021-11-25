参考 https://github.com/AobingJava/JavaFamily

## 1.基础

### 1.集合框架

#### 1.HashMap

------

​	HashMap 是由链表和数组组合而成的基本数据结构 ，k-v 存储，允许null key 和null value

​	![](https://gitee.com/lifutian66/img/raw/master/img/3688928-84c52979b3788ef7.jpg)



​	初始容量  1 << 4  **16**

​	扩容因子 **0.75f**   即 容量 * 扩容因子 为可存储做多数据，超过会扩容 成原来的  **2** 倍  增加时考虑

​	链表长度 > **8** 由 链表转换成 红黑树  增加的时候 考虑（并且 容量 > **64**  ）

​	链表长度  < **6** 由 红黑树换成 链表转  split的时候 考虑





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

   

   

   ![64撒大声地](https://gitee.com/lifutian66/img/raw/master/img/64%E6%92%92%E5%A4%A7%E5%A3%B0%E5%9C%B0.png)

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

   ![640saa](https://gitee.com/lifutian66/img/raw/master/img/640saa.png)

   写优先：

   ![smssjsj](https://gitee.com/lifutian66/img/raw/master/img/smssjsj.png)

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

![saaass](https://gitee.com/lifutian66/img/raw/master/img/saaass.png)

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

![sadadd](https://gitee.com/lifutian66/img/raw/master/img/sadadd.png)

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

![sada](https://gitee.com/lifutian66/img/raw/master/img/sada.png)

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

![虚拟机](https://gitee.com/lifutian66/img/raw/master/img/%E8%99%9A%E6%8B%9F%E6%9C%BA.png)

##### 1.详情

虚拟机内存以及dump查看工具为jdk8 bin目录下的  jvisualvm.exe

###### **1.程序计数器**

​		可以看作是当前线程所执行的 字节码的行号指示器，一个处理器（对于多核处理器来说是一个内核）都只会执行一条线程中的指令。因 

此，为了线程切换后能恢复到正确的执行位置，每条线程都需要有一个独立的程序计数器，各条线程之间计数器互不影响，独立存储，我们称这类内存

区域为“线程私有”的内存。

​		如果线程正在执行的是一个Java方法，这个计数器记录的是正在执行的虚拟机字节码指令的地址；如果正在执行的是本地（Native）方法，这个计数器值则应

为空（Undefined）

###### **2.Java虚拟机栈** 

​		虚拟机栈的生命周期与线程相同， 描述的是 java方法执行的线程内存模型，每个方法执行的时候，java虚拟机会同步创建一个栈帧，用于存储局部变量表、

操作数栈、动态链接、方法入口等信息，每个方法被调用直至执行完毕的过程，就对应着一个栈帧在虚拟机栈中入栈到出栈。

​		虚拟机栈中主要是 局部变量表，存放了编译期可知的各种Java虚拟机基本数据类型(boolean、char、byte、short、int、long、double、float)、对象引用

（reference类型，并不等同于对象本身，可能是一个指向对象起始地址的引用指针，也有可能是指向一个代表对象的句柄或者其他与此对象相关的位置）、

returnAddress类型（指向了一个字节码指令的地址）

​		线程请求的栈深度大于虚拟机允许的深度，抛出StackOverflowError 异常。

​		如果Java虚拟机栈容量可以动态扩展当栈扩展时无法申请到足够的内存会抛出OutOfMemoryError异常。

###### **3.本地方法栈**

​		本地方法栈与虚拟机栈相似，区别就是 java虚拟机栈为虚拟机执行java方法（字节码）服务，本地方法栈为虚拟机使用本地方法（native）服务

​		**HotSpot虚拟机中并不区分虚拟机栈和本地方法栈**，参数-Xoss参数虽然存在，但是在Hotspot不起作用，栈容量由-Xss决定

​		StackOverflowError  代码

​	情况1： 创建大量本地变量，占用大量局部变量表，增大栈帧长度

```java
	public static void main(String[] args) {
        try {
            test2();
        } catch (Error e) {
            System.out.println("stack length:" + stackLength);
            throw e;
        }
    }
    private static int stackLength = 0;
    /**
     * java 栈溢出
     * 1.定义大量的变量，增大本地方法栈方中本地变量表的长度。
     * VM args: -Xss128k 
     */
    public static void test2() {
        long unused1, unused2, unused3, unused4, unused5, unused6, unused7, unused8, unused9,
            unused10, unused11, unused12, unused13, unused14, unused15, unused16, unused17, unused18,
            unused19, unused20, unused21, unused22, unused23, unused24, unused25, unused26, unused27,
            unused28, unused29, unused30, unused31, unused32, unused33, unused34, unused35, unused36,
            unused37, unused38, unused39, unused40, unused41, unused42, unused43, unused44, unused45,
            unused46, unused47, unused48, unused49, unused50, unused51, unused52, unused53, unused54,
            unused55, unused56, unused57, unused58, unused59, unused60, unused61, unused62, unused63,
            unused64, unused65, unused66, unused67, unused68, unused69, unused70, unused71, unused72,
            unused73, unused74, unused75, unused76, unused77, unused78, unused79, unused80, unused81,
            unused82, unused83, unused84, unused85, unused86, unused87, unused88, unused89, unused90,
            unused91, unused92, unused93, unused94, unused95, unused96, unused97, unused98, unused99,
            unused100;
        stackLength++;
        test2();
        unused1 = unused2 = unused3 = unused4 = unused5 = unused6 = unused7 = unused8 = unused9 = unused10 =
        unused11 = unused12 = unused13 = unused14 = unused15 = unused16 = unused17 = unused18 = unused19 =
        unused20 = unused21 = unused22 = unused23 = unused24 = unused25 = unused26 = unused27 = unused28 =
        unused29 = unused30 = unused31 = unused32 = unused33 = unused34 = unused35 = unused36 = unused37 =
        unused38 = unused39 = unused40 = unused41 = unused42 = unused43 = unused44 = unused45 = unused46 =
        unused47 = unused48 = unused49 = unused50 = unused51 = unused52 = unused53 = unused54 = unused55 =
        unused56 = unused57 = unused58 = unused59 = unused60 = unused61 = unused62 = unused63 = unused64 =
        unused65 = unused66 = unused67 = unused68 = unused69 = unused70 = unused71 = unused72 = unused73 =
        unused74 = unused75 = unused76 = unused77 = unused78 = unused79 = unused80 = unused81 = unused82 =
        unused83 = unused84 = unused85 = unused86 = unused87 = unused88 = unused89 = unused90 = unused91 =
        unused92 = unused93 = unused94 = unused95 = unused96 = unused97 = unused98 = unused99 = unused100 = 0;
    }
```

​	控制台打印：

​	stack length:52

​	java.lang.StackOverflowError

情况2： 方法不断递归 占用大量栈帧

```java

    public static void main(String[] args) {
        try {
            test22();
        } catch (Error e) {
            System.out.println("stack length:" + stackLength);
            throw e;
        }
    }
    private static int stackLength = 0;
    /**
     * java 栈溢出
     * 2.方法不断递归 占用大量栈帧
     * VM args: -Xss128k
     */
    public static void test22(){
        stackLength++;
        test22();
    }
```

控制台：

stack length:1086

java.lang.StackOverflowError



###### **4.堆**

​		虚拟机启动时创建，几乎所有的对象和数组都在堆上分配内存，是垃圾回收器管理的区域。所有线程共享的java堆都可以划分出多个线程私有的线程缓冲区（Thread Local Allocation Buffer，TLAB）以提升对象分配时的效率。

OOM 示例

```java
 /**
     * java 堆溢出
     * 1.频繁创建对象，保证不被回收
     * VM Args -Xmx20m -Xms20m -XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=d:\test1.hprof
     */ 
public static void main(String[] args) {
        List<OOMObject> list = new ArrayList<OOMObject>();
        while (true) {
            list.add(new OOMObject());
        }
 }

    static class OOMObject {
    }
```

报错信息为：java.lang.OutOfMemoryError: Java heap space

jvisualvm 打开dump 详情

![image-20210428105940526](https://gitee.com/lifutian66/img/raw/master/img/image-20210428105940526.png)

以上显示对象过多创建 

查看单个对象的 GC Roots，其引用持有为ArrayList 垃圾回收根节点，无法回收，gc不回收，对象创建过多，导致OOM

![image-20210428110543704](https://gitee.com/lifutian66/img/raw/master/img/image-20210428110543704.png)

###### **5.方法区 **

​		jdk8之前  方法区称呼为“永久代”，本质上两者并不等价，只是使用永久代实现了方法区，这样方法区 就能像堆一样 做垃圾管理，jdk8废弃了永久代，采用本

​		地内存中实现元空间

​		线程共享的，用于存储，java虚拟机加载的类型信息、常量、静态变量、即时编辑器编译后的代码缓存

###### **6.运行时常量池**

​		是方法区的一部分，class 文件除了版本信息，字段，方法，接口描述信息等还有常量，用于存放编译期生成的各种字面量和符号引用，这部分内容将会在类

​		加载后存放在方法区的运行时常量池，常量池具备动态性，并非预置入Class文件中常量池的内容才能进入方法区运行时常 量池，运行期间也可以将新的常量

​		放入池中，**String类的 intern()方法**，C++的 StringTable::intern方法实现，在当前类的常量池中查找是否存在与str等值的String，若存在则直接返回常量池

​        中相应Strnig的引用；若不存在，则会在常量池中创建一个等值的String，然后返回这个String在常量池中的引用

​		**注意**：由于jdk7开始逐步去永久代计划，jdk6以及之前的虚拟机的常量池实现是在永久代，可通过参数-XX: PermSize 和-XX: MaxPermSize 调整永久代大小，

​		间接调整常量池的大小，可用过String::intern() 测试， 其异常java.lang.OutOfMemoryError: PermGen space 

​	    jdk8是通过元空间（堆）实现的常量池，其限制方法区大小测试 是毫无意义的 ，只会出现的OOM 也会提示  Java heap space 

​		元空间 参数限制 ：

​		-XX：MaxMetaspaceSize：设置元空间最大值，默认是-1，即不限制，或者说只受限于本地内存大小 

​		-XX：MetaspaceSize：设置元空间大小，单位字节，达到该值就会触发垃圾收集 进行类型卸载，同时收集器会对该值进行调整：如果释放了大量的空间，就

​				适当降低该值；如果释放了很少的空间，那么在不超过-XX：MaxMetaspaceSize（如果设置了的话）的情况下，适当提高该值

​	

###### **7.直接内存** 

​		并不是虚拟机运行时数据区的一部分，jdk1.4引入NIO，可以使用Native函数库直接分配堆外内存，然后通过一个存储在Java堆里面的 DirectByteBuffer对象

作为这块内存的引用进行操作。这样能在一些场景中显著提高性能，因为避免了 在Java堆和Native堆中来回复制数据，直接使用的是物理机内存。

​		**注意**：

​		容量大小可通过-XX：MaxDirectMemorySize 参数来指定 直接内存的最大值

​		NIO 中 DirectByteBuffer类直接通 过反射获取Unsafe实例进行内存分配

​	OOM 代码

​	

```java
   /**
	 *  无限使用直接内存
     *  VM Args -Xmx20M -XX:MaxDirectMemorySize=10M  -XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=d:\test1.hprof
     */
	private static final int _1MB = 1024 * 1024;
    public static void main(String[] args) throws IllegalAccessException {
        Field unsafeField = Unsafe.class.getDeclaredFields()[0];
        unsafeField.setAccessible(true);
        Unsafe unsafe = (Unsafe) unsafeField.get(null);
        while (true) {
            unsafe.allocateMemory(_1MB);
        }
    }
```

报错：java.lang.OutOfMemoryError 

​           at sun.misc.Unsafe.allocateMemory(Native Method) 

有时候 OOM 但是堆信息无明显错误，考虑 使用了NIO 这时候就应该考虑 直接内存 使用



##### 2.对象（HotSpot）

1. 对象组成

   由三部分组成，对象头、实例数据、对齐填充

   ![对象详情](https://gitee.com/lifutian66/img/raw/master/img/%E5%AF%B9%E8%B1%A1%E8%AF%A6%E6%83%85.jpg)

   对象头有两部分组成：

   ​	1.mark work，对象运行时数据，hashcode、GC分带年龄、锁定状态、线程池有锁、偏向锁id、偏向时间戳等，动态数据结构，空间少储存多，复用空间

   ​		未开启压缩指针的话，32系统就是32位，64就是64

   ```java
   //  32 bits:
   //  --------
   //             hash:25 ------------>| age:4    biased_lock:1 lock:2 (normal object)
   //             JavaThread*:23 epoch:2 age:4    biased_lock:1 lock:2 (biased object)
   //             size:32 ------------------------------------------>| (CMS free block)
   //             PromotedObject*:29 ---------->| promo_bits:3 ----->| (CMS promoted object)
   //
   //  64 bits:
   //  --------
   //  unused:25 hash:31 -->| unused:1   age:4    biased_lock:1 lock:2 (normal object)
   //  JavaThread*:54 epoch:2 unused:1   age:4    biased_lock:1 lock:2 (biased object)
   //  PromotedObject*:61 --------------------->| promo_bits:3 ----->| (CMS promoted object)
   //  size:64 ----------------------------------------------------->| (CMS free block)
   //
   //  unused:25 hash:31 -->| cms_free:1 age:4    biased_lock:1 lock:2 (COOPs && normal object)
   //  JavaThread*:54 epoch:2 cms_free:1 age:4    biased_lock:1 lock:2 (COOPs && biased object)
   //  narrowOop:32 unused:24 cms_free:1 unused:4 promo_bits:3 ----->| (COOPs && CMS promoted object)
   //  unused:21 size:35 -->| cms_free:1 unused:7 ------------------>| (COOPs && CMS free block)
   ```

   **hash：** 对象的哈希码

   **age：** 分代年龄

   **biased_lock：** 偏向锁标识位

   **lock：** 锁状态标识位

   **JavaThread\*：** 保存持有偏向锁的线程ID

   **epoch：** 保存偏向时间戳

   ![img](https://gitee.com/lifutian66/img/raw/master/img/20190612142936955.png)

   ![img](https://gitee.com/lifutian66/img/raw/master/img/2019061214432370.png)

   

   ​	2.类型指针，对象指向它的类型原数据指针（不是所有虚拟机都有），数组的话还需要记录数组的长度

2. 对象创建

   openjdk\hotspot\src\share\vm\interpreter\bytecodeInterpreter.cpp

   ```c++
   //new 这个指令
   CASE(_new): {
       	//获取class的下标
           u2 index = Bytes::get_Java_u2(pc+1);
       	// 获取方法区常量池
           ConstantPool* constants = istate->method()->constants();
       	//如果需要新建对象的类被解析过
           if (!constants->tag_at(index).is_unresolved_klass()) {
             // Make sure klass is initialized and doesn't have a finalizer
             Klass* entry = constants->slot_at(index).get_klass();
             assert(entry->is_klass(), "Should be resolved klass");
             Klass* k_entry = (Klass*) entry;
             assert(k_entry->oop_is_instance(), "Should be InstanceKlass");
             InstanceKlass* ik = (InstanceKlass*) k_entry;
              //确保对象所属类型已经经过初始化阶段
             if ( ik->is_initialized() && ik->can_be_fastpath_allocated() ) {
               // 取对象长度
               size_t obj_size = ik->size_helper();
               oop result = NULL;
               // 记录是否需要将对象所有字段置零值
               bool need_zero = !ZeroTLAB;
                //是否在TLAB中分配对象
               if (UseTLAB) {
                 result = (oop) THREAD->tlab().allocate(obj_size);
               }
               // Disable non-TLAB-based fast-path, because profiling requires that all
               // allocations go through InterpreterRuntime::_new() if THREAD->tlab().allocate
               // returns NULL.
   #ifndef CC_INTERP_PROFILE
               if (result == NULL) {
                 need_zero = true;
                 // 直接在eden中分配对象
               retry:
                 HeapWord* compare_to = *Universe::heap()->top_addr();
                 HeapWord* new_top = compare_to + obj_size;
               // cmpxchg是x86中的CAS指令，这里是一个C++方法，通过CAS方式分配空间，并发失败的 话，转到retry中重试直至成功分配为止
                 if (new_top <= *Universe::heap()->end_addr()) {
                   if (Atomic::cmpxchg_ptr(new_top, Universe::heap()->top_addr(), compare_to) != compare_to) {
                     goto retry;
                   }
                   result = (oop) compare_to;
                 }
               }
   #endif
               if (result != NULL) {
                 //  如果需要，为对象初始化零值
                 if (need_zero ) {
                   HeapWord* to_zero = (HeapWord*) result + sizeof(oopDesc) / oopSize;
                   obj_size -= sizeof(oopDesc) / oopSize;
                   if (obj_size > 0 ) {
                     memset(to_zero, 0, obj_size * HeapWordSize);
                   }
                 }
                  // 根据是否启用偏向锁，设置对象头信息 
                 if (UseBiasedLocking) {
                   result->set_mark(ik->prototype_header());
                 } else {
                   result->set_mark(markOopDesc::prototype());
                 }
                 result->set_klass_gap(0);
                 result->set_klass(k_entry);
                 // Must prevent reordering of stores for object initialization
                 // with stores that publish the new object.
                 OrderAccess::storestore();
                 // 将对象引用入栈，继续执行下一条指令  
                 SET_STACK_OBJECT(result, 0);
                 UPDATE_PC_AND_TOS_AND_CONTINUE(3, 1);
               }
             }
           }
   ```

   其流程大致：

   ![对象创建](https://gitee.com/lifutian66/img/raw/master/img/%E5%AF%B9%E8%B1%A1%E5%88%9B%E5%BB%BA.jpg)

3. 对象访问

   1.句柄池访问

   ​	间接访问，在堆内会划分出一块作为句柄池，reference中存储就是对象的句柄地址，句柄中包含 实例数据 和 类型数据信息的地址

   ​	好处就是在垃圾回收移动时，只需要修改句柄池指向对象实例数据的指针，增加一次指针定位开销

   ​	![image-20210428103846411](https://gitee.com/lifutian66/img/raw/master/img/image-20210428103846411.png)

   2.直接访问
   
   ​	目前机HotSpot虚拟机在用，节省一次指针定位

![image-20210428103916118](https://gitee.com/lifutian66/img/raw/master/img/image-20210428103916118.png)

双亲委派：

加载类的时候 

如果一个类加载器收到了类加载的请求，他首先不会自己去尝试加载这个类，而是把这个请求委派父类加载器去完成。每一个层次的类加载器都是如此，因此所有的加载请求最终都应该传送到顶层的启动类加载器中，只有当父加载器反馈自己无法完成这个请求（他的搜索范围中没有找到所需的类）时，子加载器才会尝试自己去加载。

![c7f1e8b004c1465ca0203bb408d6e861](https://gitee.com/lifutian66/img/raw/master/img/c7f1e8b004c1465ca0203bb408d6e861.jpeg)

- 最基础：Bootstrap ClassLoader（加载JDK的/lib目录下的类）
- 次基础：Extension ClassLoader（加载JDK的/lib/ext目录下的类）
- 普通：Application ClassLoader（程序自己classpath下的类）

破坏双亲委派：

​	SPI   JDBC  因为 Bootstrap ClassLoader  加载不到，故使用了 APPClassLoader 加载数据

#### 2.垃圾回收

##### 1.什么是垃圾

 什么是引用？

​	分为 强 软 弱 虚 

​	强引用  o=new Object() ，只要引用关系在，垃圾回收就不会回收

​	软引用  SoftReference， 生命周期 是在 发生内存溢出前后 

​	弱引用  WeakReference， 生命周期 是在 一次 垃圾回收 前后

​	虚引用 PhantomReference，作用 为了能在这个对象被收集器回收时收到一个系统通知

###### 1.引用计数

​	对象内部维护一个计数器，当被引用的时候加1，引用失效时，减一，直到没引用时变成0，即可回收

​	缺点：无法解决循环引用

###### 2.可达性分析

​	主流语言都采用此算法，基本思路就是通过一系列称为“GC Roots”的根对象最为起点集，从这些节点开始，根据这些引用关系，向下搜索，搜索过程称为

引用链，如果哪个对象 与GC Roots没有引用链，那么这个对象就 可以被回收了，参考下图

![image-20210428155347352](https://gitee.com/lifutian66/img/raw/master/img/image-20210428155347352.png)

###### 3.方法区

常量的回收 只需判断是否被引用，常量池中的接口、方法、字段的符号引用其他也是一样

判定一个类型是否属于“不再被使用的类”的条件 比较苛刻：

​	1.该类所有的实例都已经被回收（包含派生子类）

​	2.加载该类的类加载器已经被回收，这个条件除非是经过精心设计的可替换类加载器的场景，如OSGi、JSP的重加载等，否则通常是很难达成的。

​	3.该类对应的java.lang.Class对象没有在任何地方被引用，无法在任何地方通过反射访问该类的方法

以上条件 满足 被允许对回收，并不是一定会被回收 



注意：finalize() 方法不建议使用 ，try-finally或者其他方法更好

##### 2.什么时候回收

内存不足，或超过设置百分比

##### 3.怎么回收

垃圾算法：**引用计数式垃圾收集**  和  **追踪式垃圾收集**  也被称为 **直接垃圾收集** 和 **间接垃圾收集**

由于引用计数算法均为在主流垃圾回收中被采用，故以下算法均为  **间接/追踪式垃圾收集**

大多数商用虚拟机大多数遵循了 **分代收集**（人们长久积累的经验） 新生代（对象朝生晚死）+老年代（几乎不会被回收）

以分代理论为基础，大多数虚拟机均是将**java堆**分割成不同的区域，然后将对象按照其年龄（经过垃圾收集的次数）分配进入不同的区

根据分代/分区不同的特点，采用不同算法

**跨代引用** 新生代引用老年代对象，为了不在为了少量的跨代引用扫描整个老年代，在新生代建立一个全局的数据结构（“记忆集”），这个结构

把老年代划分成若干小块，标识出老年代哪块存在跨代引用，在Minor GC 时，只有包含了跨代引用的小块内存的对象才会被GC Roots 扫描

**Minor GC** 新生代收集  

**Major GC** 老年代收集  

**Mixed GC** 混合收集 新生代和 部分老年代垃圾收集  G1中会有这种情况

**Full GC** 整堆收集

###### 1.标记清除

Mark-Sweep 顾名思义 先**标记** 然后**清除**

![image-20210430134123644](https://gitee.com/lifutian66/img/raw/master/img/image-20210430134123644.png)

缺点：1.执行效率低，随着数量增长而降低

​			2.内存空间碎片化 

###### 2.标记复制

​		先标记然后复制   到预留区，将原区域清空，大多数用于 堆的新生代

​	![image-20210430134224499](https://gitee.com/lifutian66/img/raw/master/img/image-20210430134224499.png)

hotspot新生代会被划分成三块 Eden：Survivor = 8: 1 如下图

![123](https://gitee.com/lifutian66/img/raw/master/img/123.png)

每次新生代中可用内存为整个空间的90%，只有一个Survivor 是被 “浪费的”，即 10%为预留区，但是谁也不能100%保证Minor GC 要复制的对象 不超过一个 Survivor 区，那么就需要依赖其他内存区域（老年代）进行分配担保

###### 3.标记整理

先标记，后进行移动整理，老年代采用此算法 ，其考量有 移动则内存回收变复杂(移动，更新引用 需要（stop the world)，不移动内存分配变复杂(空间碎片)

吞吐量---》标记--整理  减少分配时间

低延迟---》标记--清除  减少 stop the world

另外一种就是，默认采用 标记清除，达到一定程度之后 标记整理 

![image-20210430135623696](https://gitee.com/lifutian66/img/raw/master/img/image-20210430135623696.png)

具体垃圾收集器：

![垃圾回收器-1550915446518](https://gitee.com/lifutian66/img/raw/master/img/%E5%9E%83%E5%9C%BE%E5%9B%9E%E6%94%B6%E5%99%A8-1550915446518.jpg)

**注意**：

**1.**尽可能地缩短垃圾收集时用户线程的停顿时间，即最短停顿，低延迟    CMS（增量式更新） ，G1（原始快照）《标记阶段的并发》

**2.**可控 吞吐量  处理器用于运行用户代码的时间与处理器总消耗时间的比值   Parallel Scavenge

**3.**垃圾回收 不可能三角 内存占用（Footprint）、吞吐量（Throughput）和延迟（Latency）

**4.**Shenandoah（RedHat） 和 ZGC 两个 低延迟垃圾收集器

**5.**Shenandoah 类似G1，但是抛弃分代算法，记忆集采用连接矩阵 全局数据结构 （二维表格） ，其多个阶段都是并发的，其**并发核心**  Brooks Pointer

被移动对象原有的内存上设置保护陷阱（Memory Protection Trap），一旦用户程序访问到归属于旧对象的内存空间就会产生自陷中段，进入预设好的异 

常处理器中，再由其中的代码逻辑把访问转发到复制后的新对象上，但是 用户态频繁切换到核心态 ，那么其改进为，不需要用到内存保护陷阱，而是在原有对象

布局结构的最前面统一增加一个新的引用字段，在正常不处于并发移动的情况下，该引用指向对象自己，否则指向新的复制/移动 对象，线程并发是cas，为了保证

访问一致性，引入了读、写屏障

**6.**ZGC收集器是一款基于Region内存(动态创建和销毁，以及动态的区域容量大小)布局的，（暂时） 不设分代的，使用了读屏障、染色指针和内存多重映射等技术

来实现可并发的标记-整理算法的，以低延迟为首要目标的一款垃圾收集器，其**并发核心**：染色指针技术+读写屏障 ，染色指针 直接把标记信息记在引用对象的指

针上

**7.****G1**垃圾回收器 面向全堆的，基于Region的堆内存布局，把连续的Java堆划分为多个大小相等的独立区域（Region），每一个Region都可以根据需要，扮演新生代

的Eden空间、Survivor空间，Humongous区域（大小超过Region容量的一半称为H区域，超过了整个Region容量的超级大对象， 将会被存放在N个连续的

Humongous Region之中），或者老年代空间。收集器能够对扮演不同角色的 Region采用不同的策略去处理，更具体的处理思路是让G1收集器去跟踪各个Region

里面的垃圾堆积的“价值”大小，价值即回收所获得的空间大小以及回收所需时间的经验值，然后在后台维护一个优先级列表，每次根据用户设定允许的收集停顿时

间（使用参数-XX：MaxGCPauseMillis指定，默 认值是200毫秒），优先处理回收价值收益最大的那些Region

![image-20210506104033855](https://gitee.com/lifutian66/img/raw/master/img/image-20210506104033855.png)

跨Region引用，各Region 自己维护记忆集，Key是别的Region的起始地址，Value是一个集合，里面存储的元素是卡表的索引号，也就是 卡表。

**8.**jdk 9之后 jdk log 命令 -Xlog[ : [selector  ] [ : [ output ] [ : [ decorators ] [ :output-options]]]] 

**9.**垃圾收集器参数总结 

![image-20210506145740796](https://gitee.com/lifutian66/img/raw/master/img/image-20210506145740796.png)

![image-20210506145801360](https://gitee.com/lifutian66/img/raw/master/img/image-20210506145801360.png)

#### 3.工具/命令

**基础工具**

##### 1.jps 虚拟机进程状况

jps [ options ] [ hostid ] 

![image-20210507101346462](https://gitee.com/lifutian66/img/raw/master/img/image-20210507101346462.png)

##### 2.jstat 虚拟机统计信息监视

jstat [ option vmid [interval[s|ms] [count]] ] 

选项**option** 主要分为三类：类加载、垃圾收集、运行期编译状况

![image-20210507102046504](https://gitee.com/lifutian66/img/raw/master/img/image-20210507102046504.png)

如果是本地虚拟机进程 **VMID与LVMID**  一致，如果是远程虚拟机进程，那VMID的格式应当是：[ protocol: ] [ // ]lvmid [@hostname[:port]/servername] 

参数**interval**和**count**代表**查询间隔和次数**

##### 3.jinfo Java配置信息

jinfo [ option ] pid   

jinfo 可修改配置信息 的值

##### 4.jmap 内存映像

jmap [ option ] vmid 

option 参数：

![image-20210507102722040](https://gitee.com/lifutian66/img/raw/master/img/image-20210507102722040.png)

##### 5.jhat 虚拟机堆转储快照分析

jhat dump 文件 访问 http://localhost:7000

##### 6.jstack 堆栈跟踪

jstack [ option ] vmid 

![image-20210507113813417](https://gitee.com/lifutian66/img/raw/master/img/image-20210507113813417.png)

可视化工具

VisualVM，JHSDB（jdk9），JConsole，JMC

##### 7.JHSDB 基于服务性代理的调试工具

![image-20210507140250898](https://gitee.com/lifutian66/img/raw/master/img/image-20210507140250898.png)

##### 8.JConsole Java监视与管理控制台 

过JDK/bin目录下的jconsole.exe

##### 9.VisualVM 多合-故障处理工具

过JDK/bin目录下的jvisualvm.exe

##### 10.Java Mission Control 可持续在线的监控工具 

付费的

### 5.数据结构与算法

算法必须符合的**5个条件**：1.输入2.输出3.有效4.有限5.明确

#### 1.**时间复杂度**

      O(1) 常数时间  
      O(log₂n) 次线性时间
      O(n) 线性时间  
      O(nlog₂n) 线性 对数时间
      O(n²) 平方时间 
      O(n³) 立方时间 
      O(2ⁿ)指数时间

#### 2.**算法思想**

**大问题拆成小问题处理**：

**分治法**       分类和排序    地上100张纸  按1到100 顺序排好 ==>先分类 1-10 11-20 等， 再逐步排序 

**递归法** 	  求阶乘/斐波那契函数   1.定义出口  2.反复执行的过程

**动态规划法**       在 递归/分治  基础 将已计算的结果缓存起来，复用

**穷举** ：

**迭代法**     for循环  帕斯卡三角形算法

**枚举法**	质数分解

**回溯法**	在穷举的基础上 不断否定一部分不需要验证的结果 

**贪心法**       机器学习  尝试选择最优解答，不断改进

#### 3.**基本数据结构**:

##### 1.数组 

连续的内存空间，静态数据结构，声明时 已确定其需要配大小

优点： 下标访问比较快 O(1) 

缺点：删除移动慢 O(n) 

##### 2.链表 

使用不连续内存空间存储的动态数据 

优点：增删快  O(1) 

缺点：查找慢 O(n) 

##### 3.栈

 先进后出，只能从栈顶存储数据，类似一摞盘子

##### 4.队列

先进先出，类似排队买东西

##### 5.树

非线性数据结构，由1个以及1个以上点组成，节点和节点直接不能形成回路

度数：每个节点所有子树的个数

层数：即该节点到 root节点的 个数

高度：树的最大层数

树叶或终端节点：度数为0 的节点

祖先：数根节点到本节点中间的所有节点

子孙：本节点往下追溯的任意节点

兄弟节点：有相同父节点的节点

同代：层数相同的节点

森林：n(>=0) 棵互斥数的集合

##### 6.图

描述 顶点之间 “连通与否”，应用于：算法和数据结构的最短路径搜索、拓扑排序等

图 是由 顶点和边组成的集合，通常G（V,E） 表示，其中V表示顶点组成的集合，E表示所有边组成的集合。

分为 无向图（边没有方向）  和有向图（边有方向）

##### 7.哈希表

将本身的 key通过特定的数学函数运算或使用其他的方法转成相对应的数据存储地址，可以实现快速存取和查询

避免碰撞和溢出，哈希函数尽量简单并且均匀

算法：1.除数取余  2.平方取中（平方取百、个位数为key）  3.折叠（把数据换成一串数,再把数字拆成几部分，再报他们加起来）

减少碰撞 均匀分布 算法简单

碰撞发生：1.线性探测 2.平方探测 3.再哈希 

#### 1.算法

**稳定性：**假定在待排序的记录序列中，存在多个具有相同的关键字的记录，若经过排序，这些记录的相对次序保持不变，即在原序列中，ri=rj，且ri在rj之前，而在排序后的序列中，ri仍在rj之前，则称这种排序算法是稳定的；否则称为不稳定的。

##### 冒泡排序

```java
  	/**
     * 冒泡排序  依次左右调换 一次循环把最大的放到最后
     * 时间复杂度 O(n²)
     * 最快O(n) 最慢 O(n²)	
     * 空间复杂度 O(1) 只需要一个额外空间
     * 稳定排序
     * 适用于数量少或有部分数据已排序的情况
     */
    public static void bubbleSort(int[] array) {
        int length = array.length;
        // 循环 n-1次 即可，最后一次 元素就一个，自然不比
        for (int i = 0; i < length - 1; i++) {
            // 每循环一次 最后一个元素就已经比较置换完成，之后的不需要比对
            for (int j = 0; j < length - i - 1; j++) {
                if (array[j] > array[j + 1]) {
                    int a = array[j];
                    array[j] = array[j + 1];
                    array[j + 1] = a;
                }
            }
        }
    }
```

##### 选择排序

```java
 /**
     * 选择排序  一次循环找到最大的,最后元素和那个最大的交换
     * 时间复杂度 O(n²)
     * 最快O(n²) 最慢 O(n²)
     * 空间复杂度 O(1) 只需要两个个额外空间
     * 不稳定排序（排序直接交换会使其稳定性扰乱）
     * 适用于数量少或有部分数据已排序的情况
     */
    public static void selectSort(int[] array) {
        int length = array.length;
        for (int i = 0; i < length; i++) {
            int maxIndex = 0;
            for (int j = 1; j < length - i - 1; j++) {
                if (array[j] > array[maxIndex]) {
                    maxIndex = j;
                }
            }
            int temp = array[maxIndex];
            array[maxIndex] = array[length - i - 1];
            array[length - i - 1] = temp;
        }
    }
```

##### 插入排序

```java
 /**
     * 插入排序  将数组中元素逐一与排序好的数据进行排序 进行插入,之后进行搬移
     * 时间复杂度 O(n²)
     * 最快O(n) 最慢 O(n²)
     * 空间复杂度 O(1) 只需要一个额外空间
     * 稳定排序
     * 大量数据搬移
     * 适用于部分数据已排序的情况 也适用于往已排序数据插入之后在排序
     */
    public static void insertionSort(int[] array) {
        //认为 i 之前的数组是 排好序的
        for (int i = 1; i < array.length; i++) {
            int temp = array[i];
            int j = i - 1;
            //j 为 已排好序数组最大的值的下标,依次往前寻找,只需要比这个值小的值的小标
            // 之后的数值依次往后移1位
            while (j >= 0 && temp < array[j]) {
                array[j + 1] = array[j];
                j--;
            }
            //最后 把这个值赋 给该坐标
            array[j + 1] = temp;
        }
    }
```

##### 希尔排序

```java
/**
 * 希尔排序  在插入排序的基础上,将数据按照特定间隔分成几个区,每个区都是用插入排序，再渐渐减少间隔的举例
 * 时间复杂度 O(n³/²)
 * 空间复杂度 O(1)
 * 不稳定排序 (分组使用插入算法,组与组之间会产生不稳定元素)
 * 适用于部分数据已排序的情况
 */
public static void hillSort(int[] array) {
    //一般以数组的一半长度为初始间隔
    int length = array.length / 2;
    //间隔 长度为0 结束递归
    while (length != 0) {
        //认为 i 之前的数组是 排好序的 间隔是 length
        for (int i = length; i < array.length; i++) {
            int temp = array[i];
            int j = i - length;
            //从j往前 寻找 插入的坐标，找到之后，依次后移length 位
            while (j >= 0 && temp < array[j]) {
                array[j + length] = array[j];
                j-=length;
            }
            //最后 把这个值赋 给寻找的坐标
            array[j + length] = temp;
        }
        length /= 2;
    }
}
```

##### 快速排序

```java
/**
 * 快速排序  分而治之, 在数据中虚拟个中间值，
 * 按此值分割数据（大的在右边，小的在左边）
 * 然后以同样方式处理左右两边数据
 * 时间复杂度 O(n²)  最快O(nlog2n) 最慢 O(n²)
 * 空间复杂度 O(n)  最坏  O(n)  最好O(log2n)
 * 不稳定排序
 * 目前平均时长最快的算法
 */
public static void quickSort(int[] array) {
    quick(array, 0, array.length - 1);
}

public static void quick(int[] d, int start, int end) {
    if (start < end) {
        int i = start, j = end;
        //基数就是第一个
        int base = d[i];
        while (i < j) {
            // 从右向左找小于base的数来填d[i]
            while (i < j && d[j] >= base) {
                j--;
            }
            //找到之后 填回 d[i] 这个位置
            if (i < j) {
                d[i++] = d[j];
            }
            // 从左向右找第一个大于等于base的数 来填 d[j]
            while (i < j && d[i] < base) {
                i++;
            }
            if (i < j) {
                d[j--] = d[i];
            }
        }
        //把基数填回去 位置 是 i的位置
        d[i] = base;
        //递归 i 的左边 和 右边
        quick(d, start, i - 1);
        quick(d, i + 1, end);
    }
}
```



##### 合并排序

```java
/**
 * 合并排序 叫归并排序 分治法的应用
 * 把待排序序列分为若干个子序列，每个子序列是有序的。
 * 然后再把有序子序列合并为整体有序序列.
 * 
 * 时间复杂度  最快O(nlog2n) 最慢 O(log2n)
 * 空间复杂度 O(n)
 * 稳定排序
 */
public static void mergeSort(int[] array) {
    int length = array.length;
    int half = length / 2;
    if (length > 1) {
        int[] left = Arrays.copyOfRange(array, 0, half);
        int[] right = Arrays.copyOfRange(array, half, length);
        //递归array的左半部分 右半部分
        mergeSort(left);
        mergeSort(right);
        //归并
        merge(array, left, right);
    }
}
/**
 * 把 a,b合并到 array
 */
public static void merge(int[] array, int[] a, int[] b) {
    int i = 0, aI = 0, bI = 0;
    while (aI < a.length && bI < b.length) {
        // a、b 依次遍历小的放array 左边
        if (a[aI] < b[bI]) {
            array[i++] = a[aI++];
        } else {
            array[i++] = b[bI++];
        }
    }
    //左、右 剩下的合并到右边  只可能剩下一边
    while (aI < a.length) {
        array[i++] = a[aI++];
    }
    while (bI < b.length) {
        array[i++] = b[bI++];
    }
}
```

##### 基数排序

```java
/**
 * 基数排序 又叫分配式排序 又称 桶子法
 * 最高位优先法，简称MSD法：先按k1排序分组，同一组中记录，关键码k1相等
 * 再对各组按k2排序分成子组，之后，对后面的关键码继续这样的排序分组
 * 直到按最次位关键码kd对各子组排序后。再将各组连接起来，便得到一个有序序列。
 * 
 * 最低位优先法，简称LSD法：先从kd开始排序，再对kd-1进行排序
 * 依次重复，直到对k1排序后便得到一个有序序列。
 * 
 * 时间复杂度为O (nlog(r)m)，其中r为所采取的基数，而m为堆数
 * 空间复杂度 O(nr)
 * 稳定排序
 */
public static void baseSort(int[] array, int p) {
    int length = array.length;
    //重新组合起来时 的下标
    int k = 0;
    //表示10的倍数
    int n = 1;
    //表示基数
    int m = 1;
    //数组的第一维表示可能的余数0-9
    int[][] temp = new int[10][length];
    //数组order[i]用来表示该位是i的数的个数
    int[] order = new int[10];
    while (m <= p) {
        for (int i = 0; i < length; i++) {
            //基数求余 为其二维坐标显示的 行坐标
            int lsd = ((array[i] / n) % 10);
            //纵坐标 为 order 的个数
            temp[lsd][order[lsd]] = array[i];
            //个数增加
            order[lsd]++;
        }
        //重新组合连接起来
        for (int i = 0; i < 10; i++) {
            if (order[i] != 0) {
                for (int j = 0; j < order[i]; j++) {
                    array[k] = temp[i][j];
                    k++;
                }
            }
            // 重置为零
            order[i] = 0;
        }
        n *= 10;
        k = 0;
        m++;
    }
}
```

##### 堆积树排序

待写

##### 顺序查找

 	又称线性查找，将数据一项一项的查找，时间复杂度O(n) 适合数据量少的查找

##### 二分查找

​	 将已排好序的数据队列，将数据切分在寻找 时间复杂度O(log2n)  适合已排好序的查找

##### 插值查找

​	 又称插补查找法，二分查找的改进版，按照数据位置分布，利用公式预测数据所在位置，再以二分法渐渐逼近

​	时间复杂度优于O(log2n)  适合已排好序的查找

##### 斐波那契查找

​		类似二分法，但是其分割范围并不是一半一半，而是以斐波那契级数分割，其查找次数少于二分法，平均时间复杂度O(log2n)

​		斐波那契查找比较复杂需要产生斐波那契树

##### 单向链表反转

```java
public static Node reverse(Node node) {
    Node pre = null;
    while (node != null) {
        //取出后一个
        Node oldNext = node.next;
        //后一个置为前一个 指针
        node.next = pre;
        //前指针后移
        pre = node;
        //后指针后移
        node = oldNext;
    }
    return pre;
}

static class Node {
    int data;
    Node next;

    public Node(int data) {
        this.data = data;
    }
}
```

#### 2.二叉树

​	**满二叉树**       高度为h，h>=0, 树的节点是2^h-1 

​	**完全二叉树**    一棵深度为k的有n个结点的[二叉树](https://baike.baidu.com/item/二叉树/1602879)，对树中的结点按从上至下、从左到右的顺序进行编号，如果编号为i（1≤i≤n）的结点与[满二叉树](https://baike.baidu.com/item/满二叉树/7773283)中编号为i的

​							结点在二叉树中的位置相同，则这棵二叉树称为完全二叉树。

​	**斜二叉树**  一个二叉树 完全没有左节点/右节点  （链表）

​	**严格二叉树**  一个二叉树非终端节点均有左右非空子树

```java
public static void main(String[] args) {
    BinaryTree tree = new BinaryTree();
    tree.insert(4);
    tree.insert(3);
    tree.insert(2);
    tree.insert(5);
    tree.insert(1);
    tree.insert(8);
    tree.insert(2);
    tree.insert(2);
    tree.frontOrder();
    System.out.println();
    tree.middleOrder();
    System.out.println();
    tree.backOrder();
}

static class BinaryTree {
    TreeNode root;

    /**
     * 插入
     * 删除
     * 遍历：
     * 前序遍历 根->左->右
     * 中序遍历 左->根->右
     * 后序遍历 左->右->根
     * 层序遍历 层数遍历
     */
    class TreeNode {
        TreeNode left;
        TreeNode right;
        int data;

        public TreeNode() {
        }

        public TreeNode(int data) {
            this.data = data;
        }
    }

    public BinaryTree() {
    }


    public BinaryTree(TreeNode root) {
        this.root = root;
    }

    void insert(int data) {
        TreeNode newNode = new TreeNode(data);
        if (root == null) {
            this.root = newNode;
        } else {
            TreeNode currentNode = root;
            TreeNode parentNode;
            while (true) {
                parentNode = currentNode;
                //右子树
                if (data > currentNode.data) {
                    currentNode = currentNode.right;
                    if (currentNode == null) {
                        parentNode.right = newNode;
                        return;
                    }
                } else {
                    currentNode = currentNode.left;
                    if (currentNode == null) {
                        parentNode.left = newNode;
                        return;
                    }
                }

            }

        }
    }

    void frontOrder() {
        frontOrder(this.root);
    }

    void middleOrder() {
        middleOrder(this.root);
    }

    void backOrder() {
        backOrder(this.root);
    }

    /**
     * 前序  根->左->右
     */
    void frontOrder(TreeNode node) {
        if (node != null) {
            System.out.print(node.data);
            frontOrder(node.left);
            frontOrder(node.right);
        }
    }

    /**
     * 中序  左->根->右
     */
    void middleOrder(TreeNode node) {
        if (node != null) {
            frontOrder(node.left);
            System.out.print(node.data);
            frontOrder(node.right);
        }
    }

    /**
     * 后序 左->右->根
     */
    void backOrder(TreeNode node) {
        if (node != null) {
            frontOrder(node.left);
            frontOrder(node.right);
            System.out.print(node.data);
        }
    }

}
```

#### 3.B+树

#### 4.红黑树

### 6.数据库

数据库： 物理操作系统文件或其他形式文件类型的集合

实例：后台线程和一个共享内存区域的组合

集群情况下，一个数据库对应多个实例，其他都是一一对应

#### 1.mysql

配置详解：

1. 配置文件位置

   mysql可以没有配置文件，会读取默认参数设置启动实例，mysql有几个默认读取配置文件的位置，按优先级读取，

   1 /etc/my.cnf   2  /etc/mysql/my.cnf  3/user/local/mysql/etc/my.cnf   4~/.my.cnf 

   当多个配置文件配置同一个参数，那么会以最后读取到配置文件中的配置为准

2. 数据存储位置

   datadir 默认是在 /user/local/mysql/data , 可通过执行 SHOW VARIABLES LIKE '%datadir%'; 语句查看目录


数据库是文件的集合，是依照某种数据模型组织起来并存放于二级存储器中的数据集合

![6401](https://gitee.com/lifutian66/img/raw/master/img/6401.png)

mysql 整体架构，连接池组件、管理服务和工具组件、sql接口组件、查询分析组件、优化组件、缓存组件、插件形式存储引擎，物理文件，详细如下：

![image-20210526143228334](https://gitee.com/lifutian66/img/raw/master/img/image-20210526143228334.png)

特点：用C和C ++编写

##### 1.存储引擎

各种引擎对比：

![image-20210810093941935](https://gitee.com/lifutian66/img/raw/master/img/image-20210810093941935.png)

可在数据库 information_schema 表 ENGINES  查看支持的引擎，也可执行 SHOW ENGINES; 查看支持引擎 

![image-20210810094310528](https://gitee.com/lifutian66/img/raw/master/img/image-20210810094310528.png)

###### **1.InnoDB** 存储引擎 

**为什么使用B+树？**

**磁盘往往不是严格按需读取，磁盘读取一般都会预读，预读的长度一般为页（page）的整倍数，页是计算机管理存储器的逻辑块，硬件及操作系统往往将主存和磁盘存储区分割为连续的大小相等的块，每个存储块称为一页（在许多操作系统中，页得大小通常为4k），主存和磁盘以页为单位交换数据。当程序要读取的数据不在主存中时，会触发一个缺页异常，此时系统会向磁盘发出读盘信号，磁盘会找到数据的起始位置并向后连续读取一页或几页载入内存中，然后异常返回，程序继续运行。数据库系统的设计者巧妙利用了磁盘预读原理，将一个节点的大小设为等于一个页，这样每个节点只需要一次I/O就可以完全载入 **



设计的目的面向在线事务处理

![image-20210526151036181](https://gitee.com/lifutian66/img/raw/master/img/image-20210526151036181.png)

 **1.架构**

整体：

![ ](https://gitee.com/lifutian66/img/raw/master/img/image-20210810101345928.png)

后台线程：

​	1.Master Thread 核心线程负责 将缓存中的数据异步刷新到磁盘，包括脏页的刷新、合并插入缓存

​	2.IO Thread 使用AIO,负责AIO的回调， write、read、insert buffer、log IO thread

​	3.Purge Thread  事务提交后，undo log回收

**5.7版本**

![](https://dev.mysql.com/doc/refman/5.7/en/images/innodb-architecture.png)

**8.0版本**

![image-20210526163147963](https://gitee.com/lifutian66/img/raw/master/img/image-20210526163147963.png)

分为在内存中的 和 在磁盘 中的两部分

**2.IN  MEMORY**

数据库页读取  读取先判断是否在缓存池中，命中返回，否则读取磁盘到缓存池，返回

数据库页修改  先修改缓存池中的页，以一定频率（Checkpoint）刷新到磁盘

可修改 `innodb_buffer_pool_size` 大小更改缓存池的大小，16GB内存的机器，默认是8M

可修改 `innodb_buffer_pool_instances` 大小更改缓存池的个数 默认是1

InnoDB 内存中的结构情况：

```

1.buffer pool   数据缓存区，在专用服务器上一般分配80%物理内存  ，底层实现 一个以页为元素的链表 （a linked list of pages）,使用LRU 算法管理内存,如下
	图:
	
 	从磁盘中读取页，并不是直接放入到**LRU列表**首部，而是放在LRU midpoint位置，默认就是5/8 处，可修改`innodb_old_blocks_pct`参数，调整其默认值是 37，也就是37%位置，差不多3/8处，midpoint之后是old 列表，否则就是new 列表

   midpoint 的位置 防止活跃热点数据被刷出缓存池，可以修改 `innodb_old_blocks_time`, 表示页读取到midpoint 位置前（new 列表），需要多少次

   数据库初始化时，LRU列表是空的，这些也都存在 **free list**（链表） 列表中. 当需要从缓存池中分页，首先看free list 是否有可用的空闲页，有的话，从free   

   list删除，放到LRU列表，否则淘汰LRU列表末尾的页。分配给新的页

	old 到  new   称为 Pages made young  

	new 到 old  称为   Page not made young /   not young

    sql SHOW ENGINE INNODB STATUS
    以查看当前 INNODB 状态

----------------------
BUFFER POOL AND MEMORY
----------------------
Total large memory allocated 8568832
Dictionary memory allocated 480267
Buffer pool size   512      
Free buffers       236
Database pages     270
Old database pages 0
Modified db pages  0
Pending writes: LRU 0, flush list 0, single page 0
Pages made young 0, not young 0
0.00 youngs/s, 0.00 non-youngs/s
Pages read 4949, created 265, written 1190
0.00 reads/s, 0.00 creates/s, 0.00 writes/s
Buffer pool hit rate 933 / 1000, young-making rate 0 / 1000 not 0 / 1000
Pages read ahead 0.00/s, evicted without access 0.00/s, Random read ahead 0.00/s
LRU len: 256, unzip_LRU len: 0
I/O sum[103]:cur[0], unzip sum[0]:cur[0]
​```

Buffer pool  512页  大小为  512*16k =8M

Free buffers  236页     空闲列表

Database pages  270页 表示LRU列表中的数量 

Modified db pages   0 表示**脏页**（LRU列表中页数据被修改了）的数量， 字段 OLDEST_MODIFICATION>0的

Pages made young 0     old 到  new

not young 0     new 到 old

Buffer pool hit rate  表示 缓存池命中几率，通常小于95% 可能是因为全表扫描导致的LRU列表污染

LRU len: 256, unzip_LRU len: 0  表示非压缩  压缩的页  ，LRU列表包含 unzip_LRU ，其内存分配是伙伴算法 根据大小查找，没有的话查找大的，分裂给其使用，否则继续找大的

可通过表  INNODB_BUFFER_PAGE_LRU  查看每个页的信息

数据刷新回磁盘的列表是 **flush 列表** 其中都是脏页，脏页既存在于LRU列表，也存在flush 列表	
```

![内容在周围的文本中描述。](https://gitee.com/lifutian66/img/raw/master/img/innodb-buffer-pool-list.png)



   



```
2.change buffer  当二级索引页不在buffer pool中，change buffer 记录这些DML更改，buffer pool 在以后会合并这些更改---称为merge，将buffer pool和   change buffer 刷新到磁盘--称为purge，如下图
  1.二级索引 通常是非唯一的，其插入相对随机，合并缓存的更改避免从磁盘再次大量随机访问
  2.随后刷新到磁盘 避免了直接大量内存占用，内存上change buffer 是 buffer pool的一部分，在磁盘上change buffer是系统表空间的一部分，不会影响其丢失数据
  适用对象为非唯一辅助索引
  
  insert buffer 、delete buffer、purge buffer
  
  以insert为例，插入一条数据（有一个非唯一索引），缓存中（因为唯一索引插入前，需要查是否唯一，也就会把磁盘数据读到到缓存中）以主键顺序存放，数据顺序刷进缓存
  页（成为脏页）,后续会被刷进磁盘，对于非唯一索引，其不需要将该索引页读到缓存池，当然如果在缓存池，那就插入到缓存池，不在的话就先放到insert buffer中，然后
  以一定频率进行insert buffer和 二级索引页 合并
  
  
 change buffer 缓存内容可由  innodb_change_buffering 变量控制 
 all  默认值  插入 删除标记  缓存区物理删除
 none 	  不缓存任何
 inserts  只缓存插入
 deletes  只缓存删除标记
 changes  插入、 删除标记
 purges   缓存区物理删除
 
 innodb_change_buffer_max_size  可调整change_buffer 在buffer pool中的占用内存比率 默认是  25 最大是50 ，表示可使用 buffer pool比率
 
 以下为change buffer 信息
 Ibuf: size 1, free list len 0, seg size 2, 0 merges
merged operations:
 insert 0, delete mark 0, delete 0
discarded operations:
 insert 0, delete mark 0, delete 0
 
 当有哪个词语不懂可以查  SELECT NAME, COMMENT FROM INFORMATION_SCHEMA.INNODB_METRICS WHERE NAME LIKE '%Ibuf%'
 Ibuf: change buffer 大小 单位为 页
 merged  insert：插入记录被 merged 的数量
 discarde insert ：丢弃的插入合并操作 的数量
 delete mark 删除标记
 seg size insert buffer 大小 2*16k
 
 insert buffer 的数据结构是一棵B+树，可存放在共享表空间，默认是ibdata1，也就是在表空间
 该数据被合并时机：
 1.辅助索引页被读取到缓存池（查 bitmap中是否有数据在insert buffer的 B+树上 ）
 2.insert buffer bitmap追踪该辅助页无可用空间
 3.master thread
 
 
 
```

大致过程：![](https://gitee.com/lifutian66/img/raw/master/img/ss%20(1).png)

![内容在周围的文本中描述。](https://gitee.com/lifutian66/img/raw/master/img/innodb-change-buffer.png)

```
3.Adaptive Hash Index  自适应哈希  内存中基于key的缓存，InnoDB发现查询可以从构建哈希索引中受益，根据访问评率和模式，那么就会建

 由变量 innodb_adaptive_hash_index 控制，ON 打开 OFF关闭

 使用索引前缀构建hash key
 
 自适应哈希索引功能是分区的。每个索引都绑定到一个特定的分区，每个分区都由一个单独的闩锁保护,分区个数 是innodb_adaptive_hash_index_parts
 
 控制，默认是8，最大是512
 
 只能做等值查询
 
Hash table size 2267, node heap has 1 buffer(s)
Hash table size 2267, node heap has 0 buffer(s)
Hash table size 2267, node heap has 0 buffer(s)
Hash table size 2267, node heap has 1 buffer(s)
Hash table size 2267, node heap has 1 buffer(s)
Hash table size 2267, node heap has 1 buffer(s)
Hash table size 2267, node heap has 1 buffer(s)
Hash table size 2267, node heap has 3 buffer(s)
0.00 hash searches/s, 0.00 non-hash searches/s


4.Log Buffer 其数据会被定期 刷进磁盘 默认大小为16MB innodb_log_buffer_size  我的机器上是1M
	
	innodb_flush_log_at_trx_commit  默认 1  
	
	1  日志在每次事务提交时写入并刷新到磁盘,符合ACID 
	0  每秒将日志写入并刷新到磁盘一次。未刷新日志的事务可能会在崩溃中丢失。
	2  日志在每次事务提交后写入，并每秒刷新到磁盘一次。未刷新日志的事务可能会在崩溃中丢失。
	
	innodb_flush_log_at_timeout 默认 1    写和刷新到磁盘 是每秒发生
	
  

Free buffers+ Database pages  + 自适应hash +Lock + INSERT BUFF 等 = Buffer pool size

事务提交，Log Buffer 刷新到磁盘，先写redo log ，然后在异步刷新到真正磁盘，崩溃，使用redo log 恢复，查找redolog checkpoint标记为 是写过的，
恢复只需要恢复checkpoint 之后的即可，前面的已经刷进磁盘，当缓存不够用时，缓存根据LRU回收最少使用的页，将脏页刷新到磁盘，那么这页checkpiont 标记为是写过
redo log 以块的方式保存，每个块大小为 512K，与磁盘扇区大小一样，所以，redo log 写入可以保证原子性，不需要doublewrite技术

---
LOG
---
Log sequence number          18616805
Log buffer assigned up to    18616805
Log buffer completed up to   18616805
Log written up to            18616805
Log flushed up to            18616805
Added dirty pages up to      18616805
Pages flushed up to          18616805
Last checkpoint at           18616805

后面的数字就是 Log sequence number   LSN,8字节数字，标记版本，每个页上都有LSN，redolog、Checkpoint 也有LSN 

```

**3.ON-Disk**

整体结构：

![image-20210819170359576](https://gitee.com/lifutian66/img/raw/master/img/image-20210819170359576.png)

​		**表空间** 共享表空间ibdata1、innodb在 每个表各自表空间

​		**段** 数据段、索引段、回滚段，段开始分配32页大小的碎片页，使用完这些页后，才是64个连续页申请（节省磁盘）

​		**区**  大小为  1M，默认有64个连续的页（32K 页 区2M，64K 页 区4M）

​		**页**  `innodb_page_siz` 默认16K 可调整   

​		![image-20210820180213011](https://gitee.com/lifutian66/img/raw/master/img/image-20210820180213011.png)

​		**Infimum Record** 记录该页中比任何主键值都小的值  **Supremum Record** 记录该页中比任何主键值都大的值

​	**行**  innodb 面向行存储的，支持`REDUNDANT`，`COMPACT`， `DYNAMIC`，和`COMPRESSED`

​		Innodb 支持文件格式 Antelope(旧的) =>支持REDUNDANT和COMPACT 、 Barracuda（新的）全支持

- REDUNDANT  为了兼容老版本，768 个字节的可变长度列值存储在 B 树节点内的索引记录中，其余部分存储在溢出页上

  ![image-20210820140221764](https://gitee.com/lifutian66/img/raw/master/img/image-20210820140221764.png)

  **头占6字节**

- COMPACT 

  ![image-20210820143549287](https://gitee.com/lifutian66/img/raw/master/img/image-20210820143549287.png)

  对于null 值并不会占用后面的数据存储空间，NULL标记位按二进制,下标代表字段，值1/0为该字段是否可以为null

  **头占5字节**

- DYNAMIC 

  同COMPACT 一样的行格式，但增加了增强的长期可变长度列和支持大型索引键前缀的存储能力

- COMPRESSED `Zlib`, `Lz4`, or `None`压缩

  提供相同的存储特性和功能的 `DYNAMIC`行格式，但增加了对表和索引数据压缩的支持

  DYNAMIC，COMPRESSED  对BLOB 采用完全的行溢出方式，

  **注意：**1.每行新增事务ID列和回滚指针列，分别为6字节和7字节的大小。若InnoDB表没有定义主键，每行还会增加一个6字节的rowid列

  ​			2.当数据行保存不了数据时，会保存在额外的页中 Uncompressed BLOB Page

  非压缩、动态 行溢出
  
  ![image-20210820174440111](https://gitee.com/lifutian66/img/raw/master/img/image-20210820174440111.png)
  
  压缩、动态的
  
  ![image-20210823101010403](https://gitee.com/lifutian66/img/raw/master/img/image-20210823101010403.png)
  
  

**1.table**

​	Innodb的表根据主键顺序存放，没主键 -> 非空索引 -> 自动创建 6字节大小的指针，可查_rowid 为该表主键字段的值

**2.index**

​	聚簇索引并不是单独的索引，是数据存储方式（聚簇 数据行和相邻的键紧凑的存储在一起，InnoDB（主键/第一个唯一非空索引/隐式定义主键）  数据在叶子结	点，索引在父辈节点） 主键随机id 会导致索引 变长 产生页分裂和空间碎片，InnoDB聚簇索引还是 尽量使用顺序插入 递增的主键  

​	![image-20210526095017243](https://gitee.com/lifutian66/img/raw/master/img/image-20210526095017243.png)



`InnoDB` 索引是B+树 数据结构。空间索引使用 R树

二级索引保存的数据是 主键和二级索引值，然后依据此主键在聚簇索引中查找 row，这个过程称为回表

索引构建分成三个阶段：

​	1.扫描[聚簇索引](https://dev.mysql.com/doc/refman/8.0/en/glossary.html#glos_clustered_index)，并生成索引 并将其添加到排序缓冲区 当[排序缓冲区](https://dev.mysql.com/doc/refman/8.0/en/glossary.html#glos_sort_buffer)已满时，将对条目进行排序并将其写到临时中间文件中

​	2.将一个或多个 临时中间文件 中的所有条目执行合并排序

​	3.已排序的条目插入 B+tree 中

**3.tablespaces**

1. **System Tablespace**

   `SHOW variables like 'innodb_data_file_path';`  可查看配置 ibdata1:12M:autoextend ,可自动增长ibdata1的文件

   5.7这个区域还包含doublewrite buffer+undo logs+change buffer+InnoDB Data Dictionary

   8.0 之后  change buffer+  doublewrite buffer

   8.0.20之后就只剩下 change buffer

2. **File-Per-Table Tablespaces**

    **innodb 默认** `SHOW variables like 'innodb_file_per_table'`  可查看是否开启

    单个表索引和数据存储的单独文件，`文件以表（`*`table_name`*.ibd`）命名，表文件表空间支持动态（DYNAMIC)和压缩（commpressed)行格式

3. **General Tablespaces**

    类似System Tablespace 是共享表空间，create tablespace 创建的共享表空间，可以创建于mysql数据目录外的其他表空间，可容纳多个表 

    支持所有的行格式

4. **Undo Tablespaces**

   存储undo logs，默认创建两个undo tablespace 为 sql执行 rollback 服务，默认在 data directory ，名字是undo_001` and `undo_002， undo tablespace   

   names 在data dictionary 是  innodb_undo_001` and `innodb_undo_002 

   16KB page size  ，undo tablespace file size is 10MB

    8.0.23 之后 有四个  extents （区/簇），每个最小16MB

5. **Temporary Tablespaces**

   临时表空间       --- 	会话临时表空间 和 全局临时表空间

   会话临时表空间---     用户创建的临时表 和  InnoDB 优化程序创建的内部临时表

   全局临时表空间--- 	用户创建的临时表的 rollback segments

**4.doublewrite buffer**

![image-20210813143013261](https://gitee.com/lifutian66/img/raw/master/img/image-20210813143013261.png)

​	doublewrite buffer 分两部分，一部分内存，一部分是物理磁盘的共享空间，都是2M,缓存池的脏页进行刷新时，并不直接写磁盘  

​	写入步骤如下：

​	1.memcpy函数将脏页复制到内存中的doublewrite buffer

​	2.内存中的doublewrite buffer，每次1M **顺序写入**共享表空间 的物理磁盘上

​    3.然后调用 fsync 函数，同步磁盘，离散写入

​	崩溃恢复：

​	1.从共享表空间中的 doublewrite 找到 写入页 的副本，将其复制到表空间文件

​	2.应用到redo log 



`innodb_doublewrite` =ON 双写区默认打开，除非 在 Linux 上为 Fusion-io NVMFS 会禁用，因为此硬件支持原子写入



​	innodb 正在写入页到表中，而这个页只写了一部分，比如 16k的页，只写了4k，这种情况就是**部分写失效**，partial page write, 这种情况可能会造成数据丢失

​	**doublewrite buffer  就是 当前写入页 写入磁盘前，先写入的 一个当前写页的副本**

​	保证数据可靠性，server崩溃，写入数据未完，恢复时，从doublewrite buffer 中数据是完整要写的数据页副本

​	redo只能加上旧、校检完整的数据页恢复一个脏块，不能修复坏掉的数据页

​	尽管数据写两次，但是不需要两倍的数据开销，只需要调用一次fsync()即可将大数据顺序写入

​	8.0.20 之后就单独存储在doublewrite files  之前是在 System Tablespace

**5.redo log**

​		基于磁盘的数据结构，记录的是对页的物理操作，比如：偏移量800，写 ‘1111’ 往磁盘写是一个扇区,512字节

​		在崩溃恢复期间用于纠正不完整事务写入的数据，记录事务日志      我的机器48M

​		 `innodb_log_file_size` 可调整其大小

​		`innodb_log_files_in_group` 日志组中的日志文件数  默认和推荐值为 2 

​		`innodb_log_group_home_dir` 可调整redolog 地址，否则会在 MySQL 数据目录 datadir

​		在提交事务之前刷新事务的redo_log `InnoDB` 使用[组提交](https://dev.mysql.com/doc/refman/5.7/en/glossary.html#glos_group_commit) 功能将多个刷新请求组合在一起，以避免每次提交一次刷新。使用组提交， `InnoDB`向日志文件发

​		出一次写入以对大约同时提交的多个用户事务执行提交操作，从而显着提高吞吐量。

​		默认在数据目录下，只有一个redo log group，每组下至少有2个redo log，例如ib_logfile0和ib_logfile1文件,循环方式写入

​		![image-20210819160733497](https://gitee.com/lifutian66/img/raw/master/img/image-20210819160733497.png)

**6.undo log**

​		存储在undo log segments, 5.7及以前 存储在 共享表空间 ,8.0存在单独的undo table space，undo log 与单个读写事务关联，记录逻辑日志,只能将数据库逻辑地恢复到原来的样子，undo log 另一个作用是MVCC，若记录被其他事务占用，当前事务可以通过undo log 读取之前 的版本信息

**4.日志文件**

`SHOW VARIABLES like 'datadir'` 查看数据位置

- **错误日志**            `SHOW VARIABLES like 'log_error'`    查看错误日志位置，主机名称.err

- **慢查询日志**  `SHOW VARIABLES like 'long_query_time'` 查看慢查询阈值  `show variables like 'slow_query_log_file'` 日志位置

    `show variables like 'log_output'` 默认file 生成慢日志，可以修改成table，之后会记录在mysql.slow_log 表里，这个表默认是  ENGINE=CSV

  可以修改成MyISAM

- **查询日志** 默认关闭 默认为: 主机名.log `show variables like '%general_log%'` 可查看位置和开关 

- **二进制日志**      记录全部对数据库进行修改的操作 **增、删、改**

   `SHOW BINLOG EVENTS IN 'LXAJT100952451-bin.000009'` 可查bin log中记录的全部信息

  二进制日志目的：**恢复、复制**，默认是关闭的，变量`sql_log_bin` 可查看当前是否开启，启用后性能稍微变慢

  `max_binlog_size`  单个binlog 文件大小，超出后，后缀+1 生成新的文件，默认大小为1G
  
  若当前使用引擎支持事务，所有的未提交的二进制日志记录在缓存中，缓存大小由`binlog_cache_size` 决定,默认32K，超过之后使用临时磁盘写入binlog
  
  可根据`SHOW GLOBAL STATUS` 中的 `Binlog_cache_disk_use`、`Binlog_cache_use`  记录了使用缓存写入binlog次数、使用磁盘写入binlog次数
  
  `sync_binlog` 表示缓存多少次写入binlog，默认是1，同步写磁盘的方式写binlog，仅在事务提交前写入磁盘，次数调大，服务器宕机会有部分未写入binlog
  
  `binlog_format` 决定binlog记录的内容：
  
  `STATEMENT`   记录日期的逻辑sql 语句，一些UUID RAND 函数执行结果可能不一样
  
  `ROW`  默认，记录表的行更改情况，binlog会变大，复制采用传输二进制日志方式实现，因此复制的网络开销也会增加
  
  `MIXED` 默认采用STATEMENT格式记录，一些情况下ROW
  
  各个引擎对binlog格式支持
  
  ![image-20210819145432215](https://gitee.com/lifutian66/img/raw/master/img/image-20210819145432215.png)
  
  `mysqlbinlog   --start-position=偏移量  日志文件`
  
  查看必须通过mysql提供的mysqlbinlog，`STATEMENT`  格式的就可以看到sql，而`ROW`需要添加 -v或-vv 查看更改，以下为row看到的结果更改sql
  
  ![image-20210819151711432](https://gitee.com/lifutian66/img/raw/master/img/image-20210819151711432.png)

###### **2.MyISAM** 存储引擎

5.1 以及之前版本的 默认引擎。全文索引，压缩、空间函数等  ，不支持事务、表锁、行级锁，崩溃之后 无法恢复   ，其缓冲池之缓存 索引文件，而不缓冲数据文件，其存储引擎的表是有MYD 和MYI 组成，MYD存储数据文件，MYI存储索引文件

使用： 对于只读数据，或者表比较小 可以忍受 修复操作 

存储：将表存储在两个文件中 .MYD（数据文件）  .MYI（索引文件） 为扩展名

表锁

索引：对BLOB 和TEXT 等长字段 也可以基于其前500个字符 创建索引，支持全文索引，基于分词创建的索引 ，可以延迟更新索引键，写入内存，在清理键缓冲区或者关闭表 时，对对应的索引块写入到磁盘，提高效率，但是崩溃 无法修复

压缩表： 对于只读数据、索引，采用压缩表减少 减少磁盘I/O   

性能问题：表锁

比较适合 ETL操作

###### **3.Archive引擎**

只支持SELECT 和INERT  在mysql5.1 支持 索引， 缓存所有的写 用zlib 对插入进行压缩，压缩比1:10 ，比MyISAM 表的磁盘I/O更少， 但每次查询都需要全表查询，适合日志和数据采集  

支持行级锁和专用缓冲区  在其查询返回之前 会阻止其他SELECT   以实现一致性读  也实现了批量插入在完成之前 对读操作 不可见 ，类似mvcc 和事务 是一个针对 插入和压缩做了优化的简单引擎

###### **4.Blackhole引擎**

没有任何存储机制，丢弃所有插入数据，记录Blackhole表的日志，可用于复制数据到备库，或者只是简单记录到日志，在特殊复制架构或者日志审核发挥作用

###### **5.CSV引擎**

不支持 索引，表=csv文件

###### **6.Memory引擎**

快速查询，使用memory表，重启之后只有表 没数据，非常适合存储临时数据的临时表，以及数仓中的维度表，支持hash 索引 默认也是hash索引，只支持表级锁，并发写入性能较低 不支持 TEXT 和BLOB 类型

###### **7.Infobright 引擎**

面向列存储



##### 2.lock

###### 1.共享锁排它锁

InnoDB 标准的行级锁 S锁和X锁

A [shared (`S`) lock](https://dev.mysql.com/doc/refman/8.0/en/glossary.html#glos_shared_lock) 

An [exclusive (`X`) lock](https://dev.mysql.com/doc/refman/8.0/en/glossary.html#glos_exclusive_lock) 

###### 2.意向锁

为了防止表锁和行锁冲突，产生的全表扫描，意向锁是表级锁 ，只是标记而不是真正的锁

An [intention shared lock](https://dev.mysql.com/doc/refman/8.0/en/glossary.html#glos_intention_shared_lock) (`IS`)

An [intention exclusive lock](https://dev.mysql.com/doc/refman/8.0/en/glossary.html#glos_intention_exclusive_lock) (`IX`)  

获取 A [shared (`S`) lock](https://dev.mysql.com/doc/refman/8.0/en/glossary.html#glos_shared_lock)  需要 先获取IS 或 IX

获取 An [exclusive (`X`) lock](https://dev.mysql.com/doc/refman/8.0/en/glossary.html#glos_exclusive_lock)  需要先获取 `IX`

###### 3.行锁

锁定 index record 

###### 4.间隙锁 (开区间)

索引记录之间的间隙的锁定,或者是对第一个或最后一个索引记录之前的间隙的锁定,唯一索引的不需要使用间隙锁 （不包括 多列唯一所引）,

主键/唯一索引 使用行锁

transaction A gap S-lock 和 transaction B  gap X-lock  可以在同一个 gap  如果 一条记录 purged，   transactions must be merged

间隙锁 只是为了防止 inserting to the gap ，一个事务持有间隙锁不会 排斥其他 事务持有 间隙锁，

当事务隔离级别 是  [`READ COMMITTED`](https://dev.mysql.com/doc/refman/8.0/en/innodb-transaction-isolation-levels.html#isolevel_read-committed) ，间隙锁对于 索引扫描和查找 是禁止的，仅用于外键约束检查和重复键检查

###### 5.next-key 锁  (左开右闭区间]

行锁和间隙锁的结合 ，锁定是索引记录 加上索引记录之前的间隙 

在默认隔离级别，`InnoDB`使用next-key锁定进行搜索和索引扫描，以防止产生幻 读

###### 6.插入意向锁

​         insert 操作的 gap lock，在多事务同时写入不同数据至同一索引间隙的时候，并不需要等待其他事务完成，不会发生锁等待，假设有一个记录索引包含键值4

   和7，不同的事务分别插入5和6，每个事务都会产生一个加在4-7之间的插入意向锁，获取在插入行上的排它锁，但是不会被互相锁住，因为数据行并不冲突。

###### 7.自增锁`AUTO-INC`锁

​	表级锁 `AUTO_INCREMENT`列

[`innodb_autoinc_lock_mode`](https://dev.mysql.com/doc/refman/8.0/en/innodb-parameters.html#sysvar_innodb_autoinc_lock_mode) 变量控制用于自动增量锁定的算法。它允许您选择如何在可预测的自动增量值序列和插入操作的最大并发之间进行权衡。

该变量 值有 0、1、2  分别表示 传统、连续、交错   1为默认值

0 所有的insert 语句获取一个 auto_inc lock 在语句结束时 才会释放该锁    语句级，保证了基于语句复制的安全

1 可以一次生成几个连续的值 auto_inc  lock  不会一直保持到语句的结束，只要 语句得到了相应的值后就可以提前释放锁

2 已经不是auto_inc  lock  性能是最好的 但是对于一个insert 得到的 id 可能不是连续的

###### 8.空间索引的谓词锁

##### 2.事务

###### 1.模型

InnoDB 事务模型 旨在 结合 传统的两段式锁 在**多版本数据库** 上的高效处理，不需要锁升级，InnoDB 锁信息占用资源很少

mysql提供了 两种事务型的存储引擎：InnoDB和NDB Cluster 另外还有一些第三方存储引擎也支持事务，比较知名的包括XtraDB和PBXT。

隔离性锁机制保证，持久性 原子性 redo log保证，undo log 保证一致性

###### 2.隔离级别

- READ UNCOMMITTED  未提交读

  **脏读**

- READ COMMITTED 提交读 

  每个无锁读根据自己查询时间，**一致性读取**查询快照信息，对于有锁读，只锁定索引记录，不锁定间隙，间隙锁定仅用于外键约束检查和重复键检查，会产生**不可重复读**，同一事务里的相同查询 在不同时间查询 可能结果不一致

- REPEATABLE READ 可重复读

  InnoDB 默认事务隔离级别, 无锁采用**一致性读取**（若更改根据**undo log**重构数据），同一事务查询根据第一个查询的时间为准查询快照，其快照读并不支持在同一事务中有DML 操作，会产生**幻读**，有锁读根据查询范围（行锁/  间隙锁、next key锁）锁定数据

- SERIALIZABLE 串行化 （加锁读）

  所有的普通查询会隐式转换 **有锁读**

SET TRANSACTION ISOLATION LEVEL命令来设置隔离级别

###### 3.**持久性**

事务实现：采用事务日志帮助事务提高效率，存储引擎修改表的数据时，只需要修改其内存拷贝，再把该修改行为记录到持久的硬盘上的事务日志中，而不是去修改数据本身持久到磁盘，事务日志采用的是追加方式，写日志的操作是磁盘上的一块区域内顺序I/O，事务日志持久以后，内存中被修改的数据在后台慢慢的刷回磁盘，预写式日志，修改数据需要写两次磁盘

###### 4.事务提交回滚

SET autocommit  禁用或启用当前会话的默认自动提交模式 

START TRANSACTION  /  BEGIN  开始事务

COMMIT 手动提交

ROLLBACK 回滚当前事务

###### 5.一致性非锁定读

使用版本控制查询快照，但是在同一事务中一致性非锁定读并不适用于 ddl语句，使用ddl之后，读取的都是新的结果

mvcc 版本控制，通过保存数据在某个时间点的快照来实现的，也就是说不管需要执行 多长时间，每个事务看到的数据都是一样的， 根据事务开始时间不同，每个事务对同一个表，同一时刻看到的数据可能不一样的

InnoDB mvcc 通过每行数据后面保存两个隐藏列来实现， 创建时间+过期时间（删除时间）  这里的时间 并不是真正时间 而是系统版本号，事务开始时刻的版本号，自动递增的，在REPEATABLE READ  重复读 隔离级别下，mvcc

1.SELECT      ①InnoDB 只查版本号早于（大小于等于）当前事务版本的数据行 ② 删除版本号 大于当前版本号的

2.INSERT       插入时 保存版本号

3.DELETE      删除时 保存版本号

4.UPDATE   新插入一行 并保存版本号，删除 原来的数据行，并更新版本号



###### 6.锁定读

`SELECT ... FOR SHARE`

`SELECT ... FOR UPDATE`

NOWAIT  使用`NOWAIT`从不等待获取行锁的锁定读取。查询立即执行，如果请求的行被锁定，则会失败并显示错误。

SKIP LOCKED  使用`SKIP LOCKED` 从不等待获取行锁的锁定读取。查询立即执行，从结果集中删除锁定的行。

NOWAIT、SKIP LOCKED 均是 行锁

###### 7.不同sql的锁

- SELECT ...... FROM  一致性读，读的是快照。除非隔离级别是[`SERIALIZABLE`](https://dev.mysql.com/doc/refman/8.0/en/innodb-transaction-isolation-levels.html#isolevel_serializable) （next-key 唯一索引 是行锁 ,当然也取决与sql的where条件）

- SELECT ... FOR UPDATE and SELECT ... FOR SHARE  一般情况下会使用唯一所引锁定行，释放非锁定行，但是一些情况下并不会立马释放，因为查询结果和原数据可能存会在差异性，例如 union 操作，在评估插入是否符合结果集的条件之前，将insert 插入临时表，在这种情况下，临时表中的行与原始数据中的行关系丢失，后面的行会在查询结束之后才会释放

- [`SELECT`](https://dev.mysql.com/doc/refman/8.0/en/select.html) with `FOR UPDATE` or `FOR SHARE` , [`UPDATE`](https://dev.mysql.com/doc/refman/8.0/en/update.html), and [`DELETE`](https://dev.mysql.com/doc/refman/8.0/en/delete.html) 根据是否使用唯一索引锁定不同，唯一所引锁定行，非唯一索引间隙锁或者next-key锁

- 一致性查询会忽略任何锁

###### 8.死锁检测

InnoDB 默认，当大量线程等待同一个锁时，死锁检测会导致速度减慢 ，可以使用 [innodb_dedicated_server ](https://dev.mysql.com/doc/refman/8.0/en/innodb-parameters.html#sysvar_innodb_dedicated_server)变量禁用死锁检测

wait-for graph等待图（存在回路即为死锁） ，锁竞争会采用超时策略，回滚undo log较少的事务



###### 9.事务调度

MySQL 8.0.20 之前，`InnoDB`也是采用先进先出 (FIFO) 算法来调度事务，CATS 算法仅在重锁争用情况下使用。MySQL 8.0.20 中的 CATS 算法增强使 FIFO 算法变得多余，允许将其删除。从 MySQL 8.0.20 开始，先前由 FIFO 算法执行的事务调度由 CATS 算法执行。在某些情况下，此更改可能会影响授予事务锁定的顺序，CATS 算法通过分配调度权重来优先考虑等待事务，该权重是根据事务阻塞的事务数计算的。例如，如果两个事务正在等待对同一个对象的锁定，则阻塞最多事务的事务被分配更大的调度权重。如果权重相等，则优先考虑等待时间最长的事务

##### 3.索引

###### 1.**类型：**  

**B+ Tree 索引**    按照顺序存储   用于查找和排序 高扇出性

底层实现：InnoDB B+Tree      NDB T-Tree 

优化：MyISAM使用前缀压缩技术使得索引更小      InnoDB  按照原数据格式进行存储

B-Tree索引 比较节点页的值和要查找的值 找到合适的指针进入下层子节点，这些指针实际上定义了子节点页中值的上下限  叶子结点 指针指向数据 

![image-20210525181757228](https://gitee.com/lifutian66/img/raw/master/img/image-20210525181757228.png)



适用于： 全键值、键值范围、键前缀查找（最左前缀）

**哈希索引**

基于hash表实现，只有精确匹配才会有效，针对每行数据对所有索引列计算一个hash code，为该数据的key， value为指向数据行的指针 

Memory 默认索引，非唯一索引，重复 链表存储

无法排序 范围查找 只支持等值比较  hash冲突会影响性能

InnoDB 自适应hash 就是为了查找更快  当然也可以自己维护 创建额外列  自定义hash函数 维护 索引值

**空间数据索引** R-Tree

地理数据存储  mysqlGIS 支持不完善

**全文索引**

查询文本中关键词  适用MATCH  AGAINST 操作

**其他索引**

联合索引（本质还是b+树）、覆盖索引（索引中就可查到要的信息）等等

###### 2.**优点**：

1.减少扫描数量

2.随机IO变成顺序I/O

3.避免排序和临时表

###### 3.高效使用

1.独立的列 索引列不能成为表达式/函数的一部分

2.索引列值过长 可以考虑 使用hash索引、前缀索引  select count(DISTINCT left(字段,前缀长度)/count(*)  前缀的选择性能够接近0.031，基本上就可用了

3.多列索引   复合索引 需要考虑 实际使用 情况  （考虑顺序）  AND OR 

4.聚簇索引并不是单独的索引，是数据存储方式（聚簇 数据行和相邻的键紧凑的存储在一起，InnoDB（主键/第一个唯一非空索引/隐式定义主键）  数据在叶子结点，索引在父辈节点） 主键随机id 会导致索引 变长 产生页分裂和空间碎片，InnoDB聚簇索引还是 尽量使用顺序插入 递增的主键  

​	![image-20210526095017243](https://gitee.com/lifutian66/img/raw/master/img/image-20210526095017243.png)

5.覆盖索引  索引中已经包含（覆盖）查询的数据，那么就没必要回表查询了,那么覆盖索引必须包含值 mysql 也就是   B-Tree

​	可用 延迟关联（延迟对列的查询，先使用覆盖索引） 使用覆盖索引

6.索引扫描做排序  满足最左前缀（前面字段是常量 这种也能满足），关联多表order by 字节引用字段全部为第一个表

7.压缩（前缀压缩）索引 MyISAM 默认使用前缀压缩索引减少索引的大小，默认只压缩字符串，原理：保存索引块中的第一个值，然后其他值跟这个值比较 保存	差异，第一个是 perform  第二个是 performance 那么第二个前缀压缩存储 为 7,ance MyISAM 对行指针也采用类似压缩，代价是某些操作变慢，因为每个值的    	压缩前缀都依赖前面的值，所以查找无法使用二分法，只能从头扫描，正序速度还不错，倒序就很差，在块中查找一行的操作平均需要扫描半个索引快

8.冗余和重复索引   在一个列上可重复创建多个索引，mysql会单独维护，避免创建重复索引，mysql唯一限制和主键限制都是通过索引实现的

9.索引查询会减少锁定的行， InnoDB在二级索引上使用共享（读）锁，但访问主键索引需要排他（写）锁。

##### 4.使用

**类型**

1.使用正确存储数据的**最小数据类型**   （简单数据类型占用较少磁盘、内存和CPU缓存，处理需要CPU周期更少）

2.尽量**避免nul**l  null使得 索引、索引统计和值都比较复杂，使用更多的存储空间，MyISAM 固定大小的索引变成 可变大小的

3.整数类型 TINYINT  SMALLINT MEDIUMINT INT BIGINT 8 16 24 32 64位存储 -2^(N-1)至2^(N-1)-1 无符号0-2^(N) INT(1) 和INT(20) 存储和计算都一样

4.实物(小数) FLOAT 4个字节 DOUBLE 8个字节 DECIMAL 二进制字符串 每四个字节存 9个数字（2^31 大致是10位数）

5.**字符串** 跟存储引擎相关 非定长的 需要**额外字段记录长度**，update变长 InnoDB 分裂页 MyISAM 将行拆成不同片段存储,比数字小号空间多 查询慢（MyISAM 默认对字符串使用压缩碎银 导致查询慢）

6.char 固定长度 空格填充 

7.BLOB和TEXT类型 二进制 字符 存储  没有排序规则或字符集 和 有 排序规则或字符集

8.**ENUM 实际存储为整数** ，可查询其双重属性，其排序是按 内部存储的整数， 枚举与char/varchar 关联会很慢

9.时间类型  **DATETIME** 范围大 1001年到9999年 精确到秒 存储 YYYYMMDDHHMMSS 整数 与时区无关，8个字节

​	**TIMESTAMP** 1970年到2038年 4个字节，有时区  默认设置为当前时间

10.位数据类型 mysql 将其当做字符串来处理  set 当做数字处理

schema 设计

1.**不要太多列**    服务器层和存储引擎层 通过缓冲数据拷贝数据，然后在服务器层将缓冲内容解码成各个列（MyISAM 非定长行和InnoDB），列再转行

2.**不要太多关联** 最多支出61张表 但是最好限制在12个表以内

3.**注意防止过度使用枚举**  set类似枚举

4.NULL 确实需要 也是可以的 并**非一定要避免使用NULL** 

5.ALTER TABLE 操作 mysql 执行大部分修改表结构操作：用新的结构创建个新表，把数据导入，然后删除旧表   当然也有例外： 

​				1.修改列的默认值 ALTER COLUMN 只修改.frm文件 不涉及表数据

​				2.移除一个列的AUTO_INCREMENT属性

​				3.增加、移除、更改ENUM和SET的可选值

​			也可以手替换.frm 文件

6.MyISAM 高效导入数据，先禁用索引，导入数据 再启用索引，构建索引工作延迟到数据载入后，可通过排序构建索引 速度快而且索引树碎片更少 更紧凑

​	InnoDB 也可以有类似 先删除索引  导入数据 在创建索引

### 7.Redis 

#### 1.底层编码

##### 1.  字符串

   ```c
   struct sdshdr{
       //记录buf数组已使用的字节数量
       int len;
       //记录buf中未使用的字节数量
       int free;
       //保存数据 保存文本数据或二进制数据
       char buf[];
   }
   ```

   ​	SDS字符串，区别于c的字符串

   ​    1.记录len便于查找

   ​    2.free len 便于进行内存分配

   ​    3.空间预分配，空间惰性回收，减少分配次数，杜绝内存溢出

   ​	4.保证二进制安全 buf[] 数组保存（不会因为'\0'结束读取）

##### 2. 链表

   ```c
   //链表节点
   typedef struct listNode {
       // 前置节点
       struct listNode * prev;
       // 后置节点
       struct listNode * next;
       // 节点的值
       void * value;
   }listNod;
   //链表
   typedef struct list {
       // 表头节点
       listNode * head;
       // 表尾节点
       listNode * tail;
       // 链表所包含的节点数量
       unsigned long len;
       // 节点值复制函数
       void *(*dup)(void *ptr);
       //节点值释放函数
       void (*free)(void *ptr);
       // 节点值对比函数
       int (*match)(void *ptr,void *key);
   } list;
   ```

   特点：双端、无环、长度计数、头尾指针、多态（类型不限制）

##### 3. 字典  k-v

数据库和hash键的底层实现

   ```c
   //hash表节点
   typedef struct dictEntry {
       //键
       void *key;
       // 值
       union{
           void *val;
           uint64_tu64;
           int64_ts64;
       } v;
       // 指向下个哈希表节点，形成链表，hash冲突解决 链表法
       struct dictEntry *next;
   } dictEntry;
   
   // hash 表
   typedef struct dictht {
       // 哈希表数组
       dictEntry **table;
       //哈希表大小
       unsigned long size;
       //哈希表大小掩码，用于计算索引值   总是等于size-1
       unsigned long sizemask;
       // 该哈希表已有节点的数量
       unsigned long used;
   } dictht;
   
   // 字典
   typedef struct dict {
       // 类型特定函数
       dictType *type;
       // 私有数据
       void *privdata;
       // 哈希表 两个hash表，rehash做切换使用
       dictht ht[2];
       //  记录rehash进度， 当rehash不在进行时，值为-1
       in trehashidx; 
   } dict;
   
   //类型 函数
   typedef struct dictType {
       // 计算哈希值的函数
       unsigned int (*hashFunction)(const void *key);
       // 复制键的函数
       void *(*keyDup)(void *privdata, const void *key);
   	// 复制值的函数
       void *(*valDup)(void *privdata, const void *obj);
       // 对比键的函数
       int (*keyCompare)(void *privdata, const void *key1, const void *key2);
       // 销毁键的函数
       void (*keyDestructor)(void *privdata, void *key);
       // 销毁值的函数
       void (*valDestructor)(void *privdata, void *obj);
   } dictType;
   ```

   根据键计算hash值和索引，将包含新键值对的hash节点放到hash表数组索引位置，存在的话，链址法处理

```c
hash = dict->type->hashFunction(key);
index = hash & dict->ht[x].sizemask;
```

**MurmurHash2**算法来计算键的哈希值

##### 4. 跳跃表

​	skiplist ，有序集合键的底层实现之一，大部分情况下效率和平衡树相媲美，实现更简单

​	

```c
//跳表节点
typedef struct zskiplistNode {
    // 层 根据幂次定律（越大的数出现概率越小） 随机生成一个介于1和32之间的值作为level数组的大小，就是层的高度
    struct zskiplistLevel {
        // 前进指针
        struct zskiplistNode *forward;
        // 跨度 记录两个节点之间的距离，指向null 跨度就是0
        unsigned int span;
    } level[];
    // 后退指针
    struct zskiplistNode *backward;
    // 分值
    double score;
    // 成员对象  在同一个跳跃表中，多个节点可以包含相同的分值，但每个节点的成员对象必须是唯一的。      
    // 按照分值大小进行排序，当分值相同时，节点按照成员对象的大小进行排序。
    robj *obj;
} zskiplistNode;

//跳跃表
typedef struct zskiplist {
    // 表头节点和表尾节点
    structz skiplistNode *header, *tail;
    // 表中节点的数量
    unsigned long length;
    // 表中层数最大的节点的层数
    int level;
} zskiplist
```

![image-20210827175759016](https://gitee.com/lifutian66/img/raw/master/img/image-20210827175759016.png)

##### 5. 整数集合

集合键的底层实现之一，有序、无重复的方式保存集合元素，数值过大升级操作，并不支持降级操作

```c
typedef struct intset {
    // 编码方式  INTSET_ENC_INT16  INTSET_ENC_INT32 INTSET_ENC_INT64
    uint32_t encoding;
    // 集合包含的元素数量
    uint32_t length;
    // 保存元素的数组
    int8_t contents[];
} intset;
```

##### 6. 压缩列表

​	列表键和哈希键的底层实现之一，当一个列表键只包含少量列表项，并且每个列表项要么就是小整数值，要么就是长度比较短的字符串，那么Redis就会使用压

缩列表来做列表键的底层实现。

​	节约内存，特殊编码的联系内存块组成的顺序型数据结构，每个节点可以保存一个字节数组或者整数值，添加/删除会导致连锁更新操作

#### 2.数据结构

​	redis 并不是直接面向存储结构，而是构建了对象，使用引用计数器回收内存

```c
pedef struct redisObject {
 	// 类型 数据对象
    unsigned type:4;
    // 编码 底层数据实现
    unsigned encoding:4;
    // 指向底层实现数据结构的指针
    void *ptr;
    // ...
} robj;
```
1. 字符串(string) 

   实现有 int（整数） 、embstr（embstr编码字符串,只读，修改则会变成raw）、  raw（简单动态字符串）

2. 哈希(hash)

   实现有 ziplist（压缩列表）、ht（哈希表） 

3. 列表(list)

   实现有 ziplist（压缩列表）、linkedlist（双端链表）

4. 集合(set)

   实现由intset（整数集合）、ht（哈希表）

5. 有序集合(zset)

   实现有 ziplist（压缩列表）、skiplist（有序集合，里面有个 字典）

   **注意：**

   **Redis会共享值为0到9999的字符串对象，对象会记录自己的最后一次被访问的时间，这个时间可以用于计算对象的空转时间。**

#### 3.数据库

redisServer 

```c
struct redisServer {
    
    // 服务器的数据库数量
    int dbnum;
    // 一个数组，保存着服务器中的所有数据库  默认0-15 16个
    redisDb *db;
    // ...
};

//每个数据库
typedef struct redisDb {
    // ...
    // 数据库键空间，保存着数据库中的所有键值对（多种数据结构）
    dict *dict;
    // 过期字典，保存着键的过期时间，键是指针指向某个键对象，值是long 类型整数，毫秒精度的UNIX时间戳
    dict *expires;
    // ...
} redisDb;
//客户端 客户端会在选择数据库之后，指针指向对应数据库，记录在 redisClient 的  redisDb *db;
typedef struct redisClient {
	// ...
	// 记录客户端当前正在使用的数据库
	redisDb *db;
    // ...
} redisClient;
```

##### 1.**过期处理** 

1.定时删除（对内存最友好的 但是对cpu时间不友好）  创建键的同时 创建一个timer 当时间到了 就删除 

2.惰性删除 （对cpu 时间最友好，但对内存不友好） 不删除 但是每次查询 都会判断是否过期 不过期才返回 否则删除

3.定期删除  每隔一段时间 检查数据库  删除 （对以上的折中）但是难点是 执行的时长和頻率

redis服务器实际使用的是 **后两种结合配合使用**

**RDB** 过期键处理： 过期键不保存到RDB中，载入RDB文件，主服务器不载入过期键，从服务器不论过期不过期都载入

**AOF** 过期键处理： 过期键没被惰性/定期处理不会有任何影响，被处理，AOF文件显示追加DEL命令，重新过期键不保存，主服务器主动DEL 发给从服务器

##### 2.数据库通知

订阅事件：订阅键（key-space notification）/订阅事件（key-event notification）

发送通知： 

```c
void notifyKeyspaceEvent(int type,char *event,robj *key,int dbid);

void notifyKeyspaceEvent(type, event, key, dbid);
```

#### 4.持久化

##### 1.RDB

**写入**：可以手动执行（save-阻塞  bgsave->子进程）,copy-on-write，也可以根据配置  save 选项 定期/定量 执行 保存一个压缩的全量二进制文件

**载入:**

![image-20210901142955917](https://gitee.com/lifutian66/img/raw/master/img/image-20210901142955917.png)

**save 选项**  **自动间隔性保存**

save 900 1
save 300 10
save 60 10000

以下三个条件满足一个即可触发 bgsave命令

·服务器在900秒之内，对数据库进行了至少1次修改。
·服务器在300秒之内，对数据库进行了至少10次修改。
·服务器在60秒之内，对数据库进行了至少10000次修改。

Redis服务器会维护一个 dirty计数器（距上次save/bgsave 修改的次数），一个lastsave属性（上次save/bgsave的时间戳）

Redis的服务器周期性操作函数**serverCron**默认每隔100毫秒就会执行一次，其中一项工作就是检查save选项所设置的保存条件是否已经满足

**结构：**![image-20210901145058091](https://gitee.com/lifutian66/img/raw/master/img/image-20210901145058091.png)

**查看**：od  -cx  dump.rdb

##### 2.AOF

AOF持久化是通过保存Redis服务器所执行的**写命令**来记录数据库状态的，

**写入：**需要配置打开 aof功能，启动命令增加 --appendonly yes 即可开启 aof，会将每条修改命令都记录，先写入缓冲区，根据**appendfsync配置值**，触发写入

事件，aof文件越来越大，那么就有重新功能

在redisServer ,有aof 的缓冲区

```c
struct redisServer {
    // ...
    // AOF缓冲区
    sds aof_buf;
    // ...
};
```

appendfsync 选项值：

always（总是写入aof文件（这指写入磁盘缓冲区），并完成磁盘同步）

everysec（每一秒写入aof文件（这指写入磁盘缓冲区），并完成磁盘同步）

no（写入aof文件（这指写入磁盘缓冲区），不等待磁盘同步，由操作系统决定何时同步）

**载入：**启动默认会载入

![image-20210901161321358](https://gitee.com/lifutian66/img/raw/master/img/image-20210901161321358.png)

**重写**：并不是重新现有aof 文件，因为现有的可能特别大，读取当前服务器的状态，命令为现有全部数据，后台子进程执行，为了解决重新期间导致的不一致，设置了aof重写缓冲区，当Redis服务器执行完一个写命令之后，它会同时将这个写命令发送给AOF缓冲区和AOF重写缓冲区，然后合并重新的文件和缓冲区内容

**另：**文件事件（套接字-I/O多路复用程序-文件派发器-事件处理器）

​		时间事件 （无序链表循环遍历）

时间处理调度：

![image-20210901181311213](https://gitee.com/lifutian66/img/raw/master/img/image-20210901181311213.png)

#### 5.多机

##### 1.复制



**旧版**：同步（sync）、  命令传播（command propagate）

**同步**：![image-20210902101144292](https://gitee.com/lifutian66/img/raw/master/img/image-20210902101144292.png)

**命令传播**：主服务器会将执行的写命令发送给从服务器执行

**同步（sync） 十分耗费资源，旧版断线重复复制问题**



**新版**:

PSYNC 代替 SYNC,有两种模式：完整重同步 和部分重同步

**部分重同步**：![image-20210902102002837](https://gitee.com/lifutian66/img/raw/master/img/image-20210902102002837.png)

主从机都会维护一个复制偏移量 ，主机发送命令传播都会将命令发送到 复制积压缓冲区，断线重连 主从机对比复制偏移量 ，当偏差在复制积压缓冲区的内容里，

那么就会执行在复制积压缓冲区取出缺少部分进行同步（部分重同步）,当超出这个范围，那么就执行完整重同步



**复制实现**：

​		① 设置主服务器的地址端口

​		②  建立套接字连接

​		③   发送ping 等待pong

​		④  身份验证 从服务器 设置 masterauth    主服务器设置requirepass 

​		⑤   发送从服务器的监听端口

​		⑥  同步

​		⑦  命令传播

##### 2.哨兵

主动切换，服务容错

![image-20210902103424820](https://gitee.com/lifutian66/img/raw/master/img/image-20210902103424820.png)

![image-20210902103534899](https://gitee.com/lifutian66/img/raw/master/img/image-20210902103534899.png)

![image-20210902103545506](https://gitee.com/lifutian66/img/raw/master/img/image-20210902103545506.png)

普通的redis服务器使用代码替换成sentinel 专用代码  :    端口默认26379 

创建**两个异步网络链接**：

一个是命令连接，这个连接专门用于向主服务器发送命令，并接收命令回复。

另一个是订阅连接，这个连接专门用于订阅主服务器的__ sentinel __:hello频道。 



默认10s 一次頻率发送info 到主服务器，主服务器回复自身信息+从服务器状况 

Sentinel会以每两秒一次的频率，向服务器的 __ sentinel __:hello频道发送信息

![image-20210902112155935](https://gitee.com/lifutian66/img/raw/master/img/image-20210902112155935.png)





sentinel 以每秒頻率向有命令连接的服务器发送ping命令，在该限制的时间内，没收到服务器的有效回复，sentinel会认为服务主观下线，询问其他的 sentinel ，

当收集到足够数量的**主观下线**（设置的quorum 参数），那么就会**客观下线**，进行故障转移



进行**故障转移**，需要sentinel 全部进行协作，选举出领头sentinel，并由此对下线主服务器进行故障转移每选举一次 纪元加一，每个认为客观下线的sentinel 都会

选自己，本着先到先得 ，相互发消息，最后过半的得票 的sentinel成为 领头sentinel 否则进行下一轮选举 ,头sentinel ：

1）在已下线主服务器属下的所有从服务器里面挑选出一个从服务器，并将其转换为主服务器。（网络状态好，优先级高，数据全的，排序，选第一个）
2）让已下线主服务器属下的所有从服务器改为复制新的主服务器。
3）将已下线主服务器设置为新的主服务器的从服务器，当这个旧的主服务器重新上线时，它就会成为新的主服务器的从服务器

##### 3.集群 

集群通过分片来进行数据共享，并提供复制和故障转移功能

**加入集群**：

![image-20210902142400380](https://gitee.com/lifutian66/img/raw/master/img/image-20210902142400380.png)

之后节点A会将节点B的信息通过Gossip协议传播给集群中的其他节点，让其他节点也与节点B进行握手，最终，经过一段时间之后，节点B会被集群中的所有节点认识。

**集群数据**：Redis集群通过分片的方式来保存数据库中的键值对：集群的整个数据库被分为16384个槽（slot），数据库中的每个键都属于这16384个槽的其中一个，集群中的每个节点可以处理0个或最多16384个槽。slots 表示 16384 二进制数组 也就是 16384 /8 =2048字节，当这个数组的元素上是1 ，表示该节点负责处理的槽，numslots属性则记录节点负责处理的槽的数量，也即是slots数组中值为1的二进制位的数量，节点相互的传播槽节点信息，以此更新 clusterNode里面其他节点的槽点信息

```c
 struct clusterNode {
  // ...
 unsigned char slots[16384/8]; 
 int numslots;
 // ...
};    
```

CRC16(key) & 16383 槽下标计算，其还会**保存槽和键的关系**（skiplist）,以便找到键所在槽

**重新分片**：

将槽以及槽所属的键值对 进行转移  可以在线操作，不需要下线

由redis-trib 负责执行 

1.通知目标准备导入

2.通知源节点指定slot键值对迁移

3.从源节点获得最多count个属于槽slot的键值对的键名（key name）

4.redis-trib对每个键都发送 发送一条命令到 源节点，源节点键原子地从源节点迁移至目标节点。

5.更改槽slot指派到目标节点

![image-20210902170617436](https://gitee.com/lifutian66/img/raw/master/img/image-20210902170617436.png)

迁移过程中当客户端向源节点发送一个与数据库键有关的命令，并且命令要处理的数据库键恰好就属于正在被迁移的槽时：

​	源节点会先在自己的数据库里面查找指定的键，如果找到的话，就直接执行客户端发送的命令

​	相反地，如果源节点没能在自己的数据库里面找到指定的键，那么这个键有可能已经被迁移到了目标节点，源节点将向客户端返回一个ASK错误，指引客户端转向正在导入槽的目标节点，并再次发送之前想要执行的命令。

![image-20210902170906971](https://gitee.com/lifutian66/img/raw/master/img/image-20210902170906971.png)

**复制与故障转移**

主节点（处理槽）、从节点（负责复制主，并在主下线期间时，接替主的工作）

**故障检测**

每个节点都会定期向 其他节点发送PING 以此检测是否在线，规定时间内你没接收到PONG回复，那么就标记本节点维护的该节点信息更新为 疑似下线，

集群中的各个节点会相互发送检测的各个节点状态信息，并将收到的消息发送，当一个集群过半主节点 认为 一个节点 疑似下线 ，那么这个就节点会被标记为FAIL 

并在集群中广播该信息，收到信息的都会标记此为下线FAIL 

**故障转移**

对于每个配置纪元，集群里每个负责处理槽的主节点都有一次投票的机会，而第一个向主节点要求投票的从节点将获得主节点的投票。

1.当从节点发现自己正在复制的主节点进入已下线状态时，从节点会向集群广播一条消息，要求所有收到这条消息、并且具有投票权的主节点向这个从节点投票。

2.如果一个主节点具有投票权,并且这个主节点尚未投票给其他从节点,其他点将向要求投票那么就会投出去

3.根据获取支持投票数量大于等于N/2+1 那么这个就会成为主节点，否则进新纪元，直至选出主节点

4.新的主节点接替槽处理工作，新主节点广播PONG消息，让集群感知到

### 8.消息队列

#### 1.RocketMQ

用于主题发布订阅，自实现nameServer 替代zookpeer **最终一致性**，只**保证消息被消费者消费**，但设计上允许消息被**重复消费**

![image-20210907101750696](https://gitee.com/lifutian66/img/raw/master/img/image-20210907101750696.png)

##### 1.消息路由

###### 1.初始化

NameServer 

核心代码：

   创建 NamesrvController，启动 NamesrvController

![image-20210907141555202](https://gitee.com/lifutian66/img/raw/master/img/image-20210907141555202.png)

如何创建  NamesrvController：

![image-20210907141704229](https://gitee.com/lifutian66/img/raw/master/img/image-20210907141704229.png)

启动 NamesrvController，初始化NamesrvController

![image-20210907141820480](https://gitee.com/lifutian66/img/raw/master/img/image-20210907141820480.png)

初始化做了：初始化两个定时任务(下图)，注册JVM优雅停止线程池的函数（在上图的Runtime）

![image-20210907142138443](https://gitee.com/lifutian66/img/raw/master/img/image-20210907142138443.png)

###### 2.路由注册

NamesrvController  构造函数 会创建RouteInfoManager，其内部存储全部路由信息

![image-20210907143815440](https://gitee.com/lifutian66/img/raw/master/img/image-20210907143815440.png)

1.Broker 启动时，启动定时任务，发送心跳到每个NameServer，发送RequestCode.REGISTER_BROKER

![image-20210907145134278](https://gitee.com/lifutian66/img/raw/master/img/image-20210907145134278.png)

![image-20210907145209538](https://gitee.com/lifutian66/img/raw/master/img/image-20210907145209538.png)

![image-20210907145814267](https://gitee.com/lifutian66/img/raw/master/img/image-20210907145814267.png)

2.NameServer 解析

NameServer  初始化时  this.registerProcessor();  注册  DefaultRequestProcessor（单机）/ClusterTestRequestProcessor（集群）

在 processRequest 找到处理  RequestCode.REGISTER_BROKER的，调用为 RouteInfoManager #  registerBroker

![image-20210907150110264](https://gitee.com/lifutian66/img/raw/master/img/image-20210907150110264.png)

![image-20210907150140496](https://gitee.com/lifutian66/img/raw/master/img/image-20210907150140496.png)

处理过程为：

![image-20210907151236107](https://gitee.com/lifutian66/img/raw/master/img/image-20210907151236107.png)

![image-20210907151807456](https://gitee.com/lifutian66/img/raw/master/img/image-20210907151807456.png)

创建/更新 topicQueueTable 列表

![image-20210907151925899](https://gitee.com/lifutian66/img/raw/master/img/image-20210907151925899.png)

###### 3.路由删除

1.超时删除

NamesrvController  初始化时的 定时任务1   RouteInfoManager#scanNotActiveBroker，循环查找，超时移除，更新brokerLiveTable、filterServerTable

brokerAddrTable、clusterAddrTable、topicQueueTable  类似注册

![](https://gitee.com/lifutian66/img/raw/master/img/Snipaste_2021-09-07_15-30-28.png)

​						![image-20210907161612167](https://gitee.com/lifutian66/img/raw/master/img/image-20210907161612167.png)

2.主动下线  

给每个NameServer发送， RequestCode.UNREGISTER_BROKER，服务端在 DefaultRequestProcessor  处理 RequestCode.UNREGISTER_BROKER,类似上面一

样的过程

![image-20210907181045751](https://gitee.com/lifutian66/img/raw/master/img/image-20210907181045751.png)

![image-20210907181106466](https://gitee.com/lifutian66/img/raw/master/img/image-20210907181106466.png)

###### 4.路由发现

RocketMQ路由发现是非实时的，当Topic路由出现变化后，NameServer不主动推送给客户端，而是由客户端定时拉取主题最新的路由

命令为 ：GET_ROUTEINTO_BY_TOPIC

![image-20210907182103726](https://gitee.com/lifutian66/img/raw/master/img/image-20210907182103726.png)

##### 3.消息发送

可靠同步发送、可靠异步发送、单向发送

消息生产者为 DefaultMQProducer

###### 1.生产者启动

 DefaultMQProducer#start   ->  DefaultMQProducerImpl#start

检查配置，更改生产者  InstanceName改为进程 id，根据生成clientId 在缓存/创建  MQClientInstance，将信息（DefaultMQProducerImpl）添加到 

MQClientInstance 的 生产者列表，启动 MQClientInstance

![](https://gitee.com/lifutian66/img/raw/master/img/Snipaste_2021-09-08_10-27-31.png)

###### 2.消息发送

DefaultMQProducer# send ->   DefaultMQProducerImpl # send ->  # sendDefaultImpl

方法参数说明： Message 发送的消息，CommunicationMode 发送消息模式（同步、异步、单次）、SendCallback 消息成功/异常的回调、timeout 超时时间

默认3S，先查找发送地址，之后消息发送即可（# sendKernelImpl），（同步）消息发送重试 默认是 2次，总共发3次

![image-20210909164501507](https://gitee.com/lifutian66/img/raw/master/img/image-20210909164501507.png)

查找路由信息，先找缓存，否则从NameServer拉取信息，更新缓存

![image-20210909170646378](https://gitee.com/lifutian66/img/raw/master/img/image-20210909170646378.png)

从NameServer更新路由信息，先获取锁，isDefault为测试选项，生产是false

true->测试用，使用默认topic,查到后，更新可读/可写队列为生产者队列数

false->正式环境，给NameServer发消息，GET_ROUTEINFO_BY_TOPIC，获取其返回列表，根据返回信息进行缓存更新

![image-20210909171940554](https://gitee.com/lifutian66/img/raw/master/img/image-20210909171940554.png)

更新详情，更新 brokerAddrTable，producerTable，consumerTable

![image-20210909171812060](https://gitee.com/lifutian66/img/raw/master/img/image-20210909171812060.png)

根据返回的 TopicPublishInfo 查找发送消息的队列![image-20210910143534019](https://gitee.com/lifutian66/img/raw/master/img/image-20210910143534019.png)

根据是否有故障延迟 规避不可用broker，剔除已经删了信息

![image-20210910152159661](https://gitee.com/lifutian66/img/raw/master/img/image-20210910152159661.png)

消息发送:  构建消息体( compressMsgBodyOverHowmuch 默认4K，超过即压缩 )，发送消息，执行前/后钩子函数

![image-20210909182412132](https://gitee.com/lifutian66/img/raw/master/img/image-20210909182412132.png)

消息发送，实现为 MQClientAPIImpl#sendMessage，具体如下， 消息发送

单次  RemotingClient#invokeOneway

异步  RemotingClient#invokeAsync

同步  RemotingClient#invokeSync

![image-20210910104805456](https://gitee.com/lifutian66/img/raw/master/img/image-20210910104805456.png)

对于消息发送编码为 GET_ROUTEINFO_BY_TOPIC  在netty 中注册的处理器 是 SendMessageProcessor （BrokerController #initialize --> #registerProcessor）

#sendMessage,校验broker,topic,检查队列，构建消息，消息最终存储

![image-20210910140110102](https://gitee.com/lifutian66/img/raw/master/img/image-20210910140110102.png)

##### 4.消息存储

RocketMQ主要存储的文件包括Comitlog文件、ConsumeQueue文件、IndexFile文件

将所有主题的消息存储在同一个文件中Comitlog

每个消息主题包含多个消息消费队列，每一个消息队列有一个消息文件 ConsumeQueue文件

IndexFile索引文件，其主要设计理念就是为了加速消息的检索性能，根据消息的属性快速从Commitlog文件中检索消息

###### 1.消息存储流程

消息发送为 this.brokerController.getMessageStore().putMessage(msgInner)

实现为：DefaultMessageStore # putMessage--> #asyncPutMessage

校验消息、状态、写消息

![image-20210913140746908](https://gitee.com/lifutian66/img/raw/master/img/image-20210913140746908.png)

详细的写，先处理延迟消息的topic、queueId

![image-20210913141746019](https://gitee.com/lifutian66/img/raw/master/img/image-20210913141746019.png)

获取写消息的 lastMappedFile,之后写即可  MappedFile# appendMessage -->  #appendMessagesInner

![image-20210913142213803](https://gitee.com/lifutian66/img/raw/master/img/image-20210913142213803.png)

获取写的缓存并设置当前指针位置，调用AppendMessageCallback # doAppend

![image-20210913142930778](https://gitee.com/lifutian66/img/raw/master/img/image-20210913142930778.png)

具体的写就是，获取偏移量，写入缓存即可

![image-20210913161833137](https://gitee.com/lifutian66/img/raw/master/img/image-20210913161833137.png)

######  2.存储文件 内存映射

CommitLog 文件存在 ${ROCKET_HOME}/store/commitlog/  其封装为  **MappedFileQueue**，每个文件对应的是 **MappedFile**

文件名为偏移量 ，**MappedFileQueue** 其属性 CopyOnWriteArrayList< MappedFil e> mappedFiles = new CopyOnWriteArrayList< MappedFile >()

为 **MappedFile** 的集合

![image-20210913162702500](https://gitee.com/lifutian66/img/raw/master/img/image-20210913162702500.png)

**MappedFileQueue** 查找文件 MappedFile  目前有两个方法

1.getMappedFileByTime    通过时间戳遍历循环 mappedFile.getLastModifiedTimestamp() >= timestamp

2.findMappedFileByOffset 通过偏移量查找 (offset / this.mappedFileSize) - (firstMappedFile.getFileFromOffset() / this.mappedFileSize)

![image-20210913182017392](https://gitee.com/lifutian66/img/raw/master/img/image-20210913182017392.png)

文件映射为 **MappedFile**，属性意义，以及初始化

![image-20210914095947740](https://gitee.com/lifutian66/img/raw/master/img/image-20210914095947740.png)

![image-20210914100316260](https://gitee.com/lifutian66/img/raw/master/img/image-20210914100316260.png)

MappedFile  # commit，真正提交为  # commit0，将脏页（writeBuffer）写入fileChannel, 也就是将byteBuffer中数据写入到channel

![image-20210914101443137](https://gitee.com/lifutian66/img/raw/master/img/image-20210914101443137.png)

![image-20210914101502257](https://gitee.com/lifutian66/img/raw/master/img/image-20210914101502257.png)

MappedFile  # flush  将数据持久化到磁盘

![image-20210914103340353](https://gitee.com/lifutian66/img/raw/master/img/image-20210914103340353.png)

当 transientStorePoolEnable = true ，创建 TransientStorePool 线程池，数据先写入到缓存池中内存，然后由commit线程定时将数据从该缓存池中内存复制到物理文件对应的缓存中，TransientStorePool  使用堆外内存，利用com.sun.jna.Library 类库将内存锁定

###### 3.存储文件

存储路径 ${ROCKET_HOME}/store

![image-20210914135517381](https://gitee.com/lifutian66/img/raw/master/img/image-20210914135517381.png)

1.commitlog：消息存储目录,commitlog文件默认1G ，消息组织方式

![image-20210914140257074](https://gitee.com/lifutian66/img/raw/master/img/image-20210914140257074.png)

根据offset 查找文件以及物理下标，读取对应长度的消息 （ 查找消息）

2.config：运行期间一些配置信息，主要包括下列信息
	consumerFilter.json：主题消息过滤信息
	consumerOffset.json：集群消费模式消息消费进度
	delayOffset.json：延时消息队列拉取进度
	subscriptionGroup.json：消息消费组配置信息
	topics.json：topic配置属性

3.consumequeue

![image-20210914141822275](https://gitee.com/lifutian66/img/raw/master/img/image-20210914141822275.png)

消息消费队列存储目录 消息消费的索引文件 consumequeue作为第一级目录消息主题，其下文件存储格式 mommitlog offset（8字节）+size（4字节）+tag hashcode（8字节）,单个文件默认 30w条，20字节 一条数据文件下标为其在消费组的逻辑偏移量，消息到达commitlog 之后专门线程产生消息转发任务，构建消费队列文件，ConsumeQueue 提供消息查找 1.根据下标 getIndexBuffer   2.根据时间 getOffsetInQueueByTime

![image-20210914144304198](https://gitee.com/lifutian66/img/raw/master/img/image-20210914144304198.png)

![image-20210914145313592](https://gitee.com/lifutian66/img/raw/master/img/image-20210914145313592.png)

4.index：消息索引文件存储目录

为消息建立索引机制，Hash索引  1.hash槽   2.hash冲突的链表

![image-20210914145557356](https://gitee.com/lifutian66/img/raw/master/img/image-20210914145557356.png)

IndexHead（开始、结束时间 开始、结束物理偏移量 hash槽、index 个数）  + hash槽 +index条目

槽存储：k->消息索引键  v->消息物理偏移量+前一个冲突的物理偏移量（index条目逻辑下标）

IndexFile # putKey 存储索引     indexFile # selectPhyOffset 查找索引

![image-20210914155318845](https://gitee.com/lifutian66/img/raw/master/img/image-20210914155318845.png)

![image-20210914155749833](https://gitee.com/lifutian66/img/raw/master/img/image-20210914155749833.png)

5.abort：如果存在abort文件说明Broker非正常关闭，该文件默认启动时创建，正常退出之前删除

6.checkpoint：文件检测点，存储commitlog文件最后一次刷盘时间戳、consumequeue最后一次刷盘时间、index索引文件最后一次刷盘时间戳。

记录commitlog,consumequeue,index文件刷盘时间点，固定为4K，只有该文件的前24字节

![image-20210914155956370](https://gitee.com/lifutian66/img/raw/master/img/image-20210914155956370.png)

###### 4.更新消费队列索引文件

commit log文件更新后，及时更新consumequeue和index 文件 该服务是在  BrokerStartup# start = BrokerController #start -->MessageStore # start -->

ReputMessageService # run   reputMessageService 设置一个初始偏移量 （ 允许重复 =CommitLog的提交指针 不允许重复=Commitlog的内存中最大偏移量）

![image-20210914162347930](https://gitee.com/lifutian66/img/raw/master/img/image-20210914162347930.png)

内部实现为，根据偏移量循环读取，每条都会 CommitLogDispatcherBuildConsumeQueue、CommitLogDispatcherBuildIndex  的 dispatch方法

![image-20210914173335435](https://gitee.com/lifutian66/img/raw/master/img/image-20210914173335435.png)

CommitLogDispatcherBuildConsumeQueue # dispatch  -->  DefaultMessageStore #  putMessagePositionInfo

根据主题和队列id找到消费的队列，之后调用队列的 putMessagePositionInfoWrapper--> putMessagePositionInfo

![image-20210914173802767](https://gitee.com/lifutian66/img/raw/master/img/image-20210914173802767.png)

![image-20210914175248472](https://gitee.com/lifutian66/img/raw/master/img/image-20210914175248472.png)

CommitLogDispatcherBuildIndex  #  dispatch    根据 isMessageIndexEnable = true 开启 否则 忽略   DefaultMessageStore. IndexService  #  buildIndex

先获取/创建 indexFile，之后放入hash索引即可，也是异步刷新到盘中

![image-20210914180245454](https://gitee.com/lifutian66/img/raw/master/img/image-20210914180245454.png)

###### 5.消息队列与索引恢复

根据是否有 abort文件生成，判断是否是正常关闭，进行恢复

![image-20210914181619227](https://gitee.com/lifutian66/img/raw/master/img/image-20210914181619227.png)

commitlog读取目录下文件，文件大小校验，设置指针为文件大小

ConsumeQueue读取目录下文件，获取该Broker存储的所有主题、队列，初始加载

Checkpoint 加载存储检测点，检测点主要记录commitlog文件、Consumequeue文件、Index索引文件的刷盘点

index 加载索引文件，如果上次异常退出，而且索引文件上次刷盘时间小于该索引文件最大的消息时间戳该文件将立即销毁。

恢复： 根据Broker是否是正常停止执行不同的恢复策略

1.先恢复 consumeque 文件，把不符合的 consueme 文件删除，一个 consume 条目正确的标准（commitlog偏移量 >0 size > 0）[从倒数第三个文件开始恢复]

2.如果 abort文件存在，此时找到第一个正常的 commitlog 文件，然后对该文件重新进行转发，依次更新 consumeque,index文件

###### 6.刷盘

broker 设置 flushDiskType，ASYNC_FLUSH（异步刷盘）、SYNC_FLUSH（同步刷盘）

![image-20210915105805283](https://gitee.com/lifutian66/img/raw/master/img/image-20210915105805283.png)

同步刷盘：GroupCommitService   MappedByteBuffer#force,调用 mappedFileQueue#flush(0) 验证是否ok

异步刷盘：未开启内存字节缓冲区 -->FlushRealTimeService  #run 调用 mappedFileQueue#flush

​					开启内存字节缓冲区    -->CommitRealTimeService # run commit成功 调用  flushCommitLogService（根据配置同步异步选择）

###### 7.定期删除

非当前写文件在一定时间间隔未更新，会被认为是过期文件，可以被删除，默认间隔时间为72小时，其实现是在

DefaultMessageStore# cleanFilesPeriodically -> CleanCommitLogService #run + CleanConsumeQueueService#run

默认10s（cleanResourceInterval）执行一次

CleanCommitLogService #run --> MappedFileQueue#deleteExpiredFileByTime  # redeleteHangedFile  删除过期文件，清除 mappedFileQueue中过期的

##### 5.消息消费

基于**拉**模式，消息消费以组的模式开展，一个消费组内可以包含多个消费者，每一个消费组可订阅多个主题

消费组之间有**集群**模式与**广播**模式两种消费模式

集群模式，主题下的同一条消息只允许被其中一个消费者消费

广播模式，主题下的同一条消息将被集群内的所有消费者消费一次

支持**局部顺序**消息消费，也就是保证**同一个消息队列**上的消息顺序消费。不支持消息全局顺序消费（可设置队列数为1 牺牲高可用）

支持两种消息**过滤模式**：表达式（TAG、SQL92）与类过滤模式

###### 1.负载与重新发布

rocket 提供**推**和**拉**两种方式供消息消费，**推模式**  实现类为 DefaultMQPushConsumer#start-->DefaultMQPushConsumerImpl#start 如下：

![image-20210916153701458](https://gitee.com/lifutian66/img/raw/master/img/image-20210916153701458.png)

其实不管消费者还是生产者其启动都是调用MQClientInstance#start ，调用 **PullMessageService**#start 拉取消息，从拉取消息队列获取一个拉去任务，执行pullMessage，没有拉去任务 将会阻塞到可获取任务为止，那么pullRequestQueue 中的PullRequest 是怎么来的呢？

![image-20210916180227300](https://gitee.com/lifutian66/img/raw/master/img/image-20210916180227300.png)

实际上 是  PullMessageService  # executePullRequestLater---> #executePullRequestImmediately

![image-20210917103735303](https://gitee.com/lifutian66/img/raw/master/img/image-20210917103735303.png)

调用这个方法的地方是：

1.DefaultMQPushConsumerImpl#pullMessage 也就是上面获取pullRequest 之后调用的方法

2.RebalancePushImpl#dispatchPullRequest  这个是消息队列负载 实现，调用为 DefaultMQPushConsumerImpl 推模式的下会调用

![image-20210917104249830](https://gitee.com/lifutian66/img/raw/master/img/image-20210917104249830.png)

**拉**模式只需要提供api即可，消费端获取客户端返回信息 实现为 MQClientInstance # start --> PullMessageService #pullMessage 其实上面已经解释了

由DefaultMQPushConsumerImpl #pullMessage  调用过来，**消息限流**(消息总数>100  最大间距>2000  消息大小>100M)，构建消息，最后发送

![image-20210917152714874](https://gitee.com/lifutian66/img/raw/master/img/image-20210917152714874.png)

之后便是消息内容构建，最后发送

 ![image-20210917152852722](https://gitee.com/lifutian66/img/raw/master/img/image-20210917152852722.png)

PullAPIWrapper#pullKernelImpl

###### 2.消费模式



###### 3.拉取



###### 4.定时消息

###### 5.消息过滤

###### 6.顺序消息



##### 6.消息过滤



##### 7.主从

##### 8.事务消息

##### 9.实战

#### 2.RabbitMq



#### 3.Kafka

### 9.分布式

#### 1.Zookpeer

##### 1.分布式历史

| 架构   | 机器     | 特点                                                     | 缺点                                             |
| ------ | -------- | -------------------------------------------------------- | ------------------------------------------------ |
| 集中式 | 大型主机 | 成本高，部署简单，性能稳定性好，基本都是单点处理         | 出了故障就是不可用                               |
| 分布式 | pc机     | 多台计算机空间分布随意，多副本，高效协调分布式并发是问题 | 缺乏全局时钟，通信异常，网络分区，三态，节点故障 |

##### 2.基础理论

事务的特点 ： ACID，原子性、一致性、隔离性、持久性

**CAP**：一个分布式系统不可能同时满足**一致性**、**可用性** 和 **分区容错性**

![image-20211123104815672](https://gitee.com/lifutian66/img/raw/master/img/image-20211123104815672.png)

**BASE**: 基本可用、软状态、最终一致性

##### 3.一致性协议

**2pc**:  提交事务请求 (协调者事务询问->执行事务->各参与者反馈事务结果 给协调者) 

​		  执行事务提交（各参与者根据协调者反馈结果执行 commit/rollback）

​		 **优缺点**：原理简单，实现方便，同步阻塞、单点问题、脑裂、保守

![image-20211123150859140](https://gitee.com/lifutian66/img/raw/master/img/image-20211123150859140.png)

![image-20211123150911898](https://gitee.com/lifutian66/img/raw/master/img/image-20211123150911898.png)

**3pc**：

CanCommit   事务询问   各参与者反馈

PreCommit  发送预提交   事务预提交    各参者反馈结果

doCommit  发送提交请求 事务提交/回滚

**优缺点**： 网络分区 造成事务不一致

![image-20211123152515144](https://gitee.com/lifutian66/img/raw/master/img/image-20211123152515144.png)

##### 4.paxos

1. 第一阶段：Prepare阶段。Proposer向Acceptors发出Prepare请求，Acceptors针对收到的Prepare请求进行Promise承诺。

2. 第二阶段：Accept阶段。Proposer收到多数Acceptors承诺的Promise后，向Acceptors发出Propose请求，Acceptors针对收到的Propose请求进行Accept处理。

3. 第三阶段：Learn阶段。Proposer在收到多数Acceptors的Accept之后，标志着本次Accept成功，决议形成，将形成的决议发送给所有Learners。

   Paxos算法流程中的每条消息描述如下：

   - **Prepare**: Proposer生成全局唯一且递增的Proposal ID (可使用时间戳加Server ID)，向所有Acceptors发送Prepare请求，这里无需携带提案内容，只携带Proposal ID即可。
   - **Promise**: Acceptors收到Prepare请求后，做出“两个承诺，一个应答”。

   **两个承诺**：

   1. 不再接受Proposal ID小于等于（注意：这里是<= ）当前请求的Prepare请求。

   2. 不再接受Proposal ID小于（注意：这里是< ）当前请求的Propose请求。

   **一个应答**：

   不违背以前作出的承诺下，回复已经Accept过的提案中Proposal ID最大的那个提案的Value和Proposal ID，没有则返回空值。

![preview](https://gitee.com/lifutian66/img/raw/master/img/v2-a6cd35d4045134b703f9d125b1ce9671_r.jpg)

#### 2.Raft

### 10.框架

#### 1.spring cloud

#### 2.doubb





