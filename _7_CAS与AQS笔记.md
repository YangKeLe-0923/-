# CAS与AQS核心知识速览

## 一、CAS是什么

**定义** **CAS = Compare And Swap（比较并交换）**，是一种无锁原子操作，用于实现多线程环境下的变量更新。

它本质上就是一种**"乐观锁"**——假设没有冲突，先去改，改的时候发现被别人改过了就重试，直到成功为止。

打个比方：CAS就像超市换货

- 你拿了一瓶旧饮料（旧值），想去换一瓶新的（新值）
- 到了柜台，先看货架上是不是还是你那瓶旧饮料（比较）
- 如果还是旧的，直接换成新的（交换），成功
- 如果已经被别人换成别的了，就再等下一次机会，重新来

### CAS三要素

- **V**：要更新的变量（内存地址）
- **A**：预期的旧值（期望V现在等于A）
- **B**：要设置的新值

操作逻辑：如果 V == A，就把 V 设为 B，返回成功；否则什么都不做，返回失败。整个操作是**原子的**，由CPU指令保证。

## 二、原子类（Atomic系列）

### 是什么

J.U.C（java.util.concurrent）包下的一系列原子操作类，底层都是用CAS实现的，用来解决多线程下的计数等问题，比synchronized更轻量。

常用原子类：

- `AtomicInteger` —— 原子整数，最常用
- `AtomicLong` —— 原子长整数
- `AtomicBoolean` —— 原子布尔值
- `AtomicReference` —— 原子引用（原子更新对象）
- `AtomicIntegerArray` —— 原子数组

### 代码示例：AtomicInteger计数器

```java
public class AtomicDemo {
    // 原子整数，初始值0
    private static AtomicInteger count = new AtomicInteger(0);

    public static void main(String[] args) throws InterruptedException {
        for (int i = 0; i < 10; i++) {
            new Thread(() -> {
                for (int j = 0; j < 1000; j++) {
                    count.incrementAndGet();  // 原子自增，不会丢数
                }
            }).start();
        }
        Thread.sleep(2000);
        System.out.println("最终结果：" + count.get());  // 一定是10000
    }
}
```

换成普通int加volatile，结果会小于10000，因为i++不是原子的。用AtomicInteger就没问题，底层CAS保证。

### ABA问题

CAS有个经典问题：变量从A变成B，又变回A。CAS检查的时候发现还是A，就以为没变过，但实际上中间已经变过了。

大部分场景ABA不影响，但某些场景（比如链表操作）会出问题。

**解法**：加版本号。用 `AtomicStampedReference`，每次修改版本号+1，比较的时候连版本号一起比。

## 三、AQS是什么

**定义** **AQS = AbstractQueuedSynchronizer（抽象队列同步器）**，是J.U.C中几乎所有锁和同步工具的基础框架。

它本质上就是一个**"排队叫号机"**——线程来了先试试能不能拿到锁，拿不到就领个号排队等着，锁释放了叫下一个号。

打个比方：AQS就像银行叫号系统

- 同步状态（state）= 窗口空闲状态，0表示没人用，大于0表示有人在用
- CLH队列 = 等候区，抢不到锁的线程排在这里
- 每个排队的线程是一个Node节点，记录着线程信息和等待状态
- 锁释放时，叫醒队列里的第一个人（队头）去抢锁

### AQS的核心思想

- **state变量**：用一个volatile int变量表示同步状态，0=空闲，>0=已占用
- **CAS改state**：用CAS方式修改state，成功就是抢到锁了
- **CLH队列**：抢不到锁的线程打包成Node节点，加入队列尾部，然后挂起（park）
- **唤醒机制**：释放锁时，把state改回0，然后叫醒队列里的下一个线程（unpark）

## 四、AQS的两种模式

### 独占模式（Exclusive）

同一时间只能有一个线程持有锁。典型代表：ReentrantLock、synchronized。

就像卫生间，一次只能进一个人。

### 共享模式（Share）

同一时间可以有多个线程持有锁，有个数量上限。典型代表：Semaphore、CountDownLatch、ReadWriteLock的读锁。

就像停车场，有N个车位，来一辆车state减1，车位满了就等，走一辆state加1。

### 自定义同步器需要实现的方法

AQS是个抽象类，子类需要实现以下方法（AQS把模板都写好了，你只需要填核心逻辑）：

- `tryAcquire(int)` —— 独占方式尝试获取锁
- `tryRelease(int)` —— 独占方式尝试释放锁
- `tryAcquireShared(int)` —— 共享方式尝试获取
- `tryReleaseShared(int)` —— 共享方式尝试释放
- `isHeldExclusively()` —— 判断是否是当前线程持有锁

你不用管排队、挂起、唤醒这些复杂逻辑，AQS都帮你做了。你只要告诉它"什么叫抢到了锁、什么叫释放了锁"就行。

## 五、AQS常见实现类

### ReentrantLock（可重入锁）

独占模式的实现。state表示锁被持有次数，同一个线程多次获取就累加，释放就递减，减到0才真正释放。

支持公平锁和非公平锁：非公平锁上来就先CAS抢一下，抢不到再排队；公平锁老老实实排队。

### Semaphore（信号量）

共享模式的实现。state表示许可数量，acquire()减1，release()加1，减到0就阻塞。

用来控制同时访问资源的线程数量，比如限流。

### CountDownLatch（倒计时门闩）

共享模式的实现。state表示剩余计数，countDown()减1，await()等待state变成0。

一个线程等多个线程都干完了再继续。

### CyclicBarrier（循环栅栏）

和CountDownLatch类似，但可以重复使用。所有线程到齐了再一起出发。

### ReentrantReadWriteLock（读写锁）

读锁共享、写锁独占。读读不互斥，读写/写写互斥。读多写少场景性能好。

## 六、为什么需要CAS和AQS

### 问题1：synchronized太重了

synchronized早期是重量级锁，抢不到就阻塞，阻塞就要操作系统切换线程上下文，开销很大。

CAS的解决方案：用CPU指令保证原子性，抢不到就自旋重试（忙等），不用阻塞线程，**用户态就能搞定**，性能好很多。

### 问题2：各种锁重复造轮子

没有AQS的话，每个同步工具都要自己实现"抢锁、排队、挂起、唤醒"这一套逻辑，代码重复，还容易写错。

AQS的解决方案：把排队、挂起、唤醒这些通用逻辑封装成模板，子类只需要填最核心的"什么是获取、什么是释放"，**大大减少重复代码**。

### 问题3：synchronized功能太单一

synchronized只有独占锁一种模式，不支持公平锁、不支持超时、不支持中断、不支持共享模式。

AQS的解决方案：基于AQS可以实现各种同步工具——可重入锁、信号量、倒计时、读写锁、栅栏等等，功能丰富得多。

## 知识点总结

1. CAS=比较并交换，乐观锁，CPU指令保证原子性，无锁但线程安全。

2. CAS三要素：内存值V、预期旧值A、新值B，V==A才把V改成B。

3. 原子类（AtomicInteger等）底层都是CAS，解决计数等场景的线程安全。

4. CAS有ABA问题，加版本号（AtomicStampedReference）可以解决。

5. AQS=抽象队列同步器，是J.U.C锁和同步工具的基础框架。

6. AQS就像排队叫号机：抢不到锁就进CLH队列排队，等前面的人释放了再叫你。

7. AQS两种模式：独占（ReentrantLock）、共享（Semaphore/CountDownLatch）。

8. AQS是模板方法模式，子类只需要实现tryAcquire/tryRelease等几个核心方法。
