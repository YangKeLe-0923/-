# JMM与volatile核心知识速览

## 一、什么是JMM

**定义** **JMM = Java Memory Model（Java内存模型）**，是Java规范中定义的一套内存访问规则，用来解决多线程环境下的可见性、有序性和原子性问题。

它本质上就是一套**"车间公告板制度"**——规定了工人（线程）什么时候看公告板（主内存）、什么时候用自己的小黑板（工作内存），以及什么时候必须把小黑板上的内容同步到公告板上。

打个比方：JMM就像办公室的白板制度

- 主内存是办公室中央的大白板，所有人都能看到，存的是共享数据
- 每个线程（员工）有自己的小笔记本（工作内存），操作数据先抄到自己笔记本上改
- 改完之后什么时候把结果写回大白板？这就是JMM管的事

## 二、三大核心问题

### 问题1：可见性

一个线程修改了共享变量的值，其他线程能不能立刻看到最新值？答案是**不能保证**。

原因：每个线程有自己的工作内存（CPU缓存），修改是先改自己的小本本，什么时候同步回主内存是不确定的。

打个比方：员工A在自己笔记本上改了数据，但没抄回白板，员工B看白板还是旧数据。

### 问题2：有序性

代码的执行顺序不一定和你写的顺序一样——编译器和CPU可能会对指令进行**重排序**来优化性能。

单线程下没问题，重排序后结果一致。但多线程下可能导致另一个线程看到的状态和预期不一样。

打个比方：做饭时你说"先淘米再洗菜"，CPU可能觉得"反正都是准备工作，先洗菜再淘米也行"，单线程没问题，但如果另一个人等着你的米下锅就乱了。

### 问题3：原子性

一个操作要么全部执行完要么不执行，中间不能被打断。比如 `i++` 看上去是一步，实际是三步：读i、加1、写回i。

多线程下这三步中间可能被其他线程打断，导致结果不对。

打个比方：两个人同时往一个存钱罐里放硬币，你拿起来数了数是10个，刚要放第11个，另一个人也拿起来数了也是10个，也放了一个写回11——本来应该12个，结果变成11个。

## 三、volatile关键字

### 是什么

**定义** **volatile** 是Java的一个关键字，用来修饰共享变量，保证变量的**可见性**和**有序性**，但**不保证原子性**。

打个比方：volatile就像车间里的**大喇叭**——一个人改了数据，大喇叭立刻广播给所有人，大家都知道最新值了。但大喇叭不能保证两个人不会同时去改同一个东西。

### 两大作用

- **保证可见性**：写volatile变量时，立刻把值刷回主内存；读volatile变量时，立刻从主内存重新读。
- **禁止指令重排序**：volatile变量前后的指令不能乱序，相当于加了一道"内存屏障"。

### 不保证原子性

volatile只能保证读和写是原子的，但 `i++` 这种"读-改-写"的复合操作不行。多线程下volatile修饰的变量做i++仍然会丢数。

需要原子性请用 `AtomicInteger` 等原子类或加锁。

### 代码示例：可见性演示

```java
public class VolatileDemo {
    // 去掉volatile，程序不会停止（线程看不到flag的更新）
    private static volatile boolean flag = true;

    public static void main(String[] args) throws InterruptedException {
        new Thread(() -> {
            while (flag) {  // 一直读flag
                // 空转
            }
            System.out.println("线程退出了");
        }).start();

        Thread.sleep(1000);
        flag = false;  // 主线程改flag
        System.out.println("主线程把flag改成false了");
    }
}
```

去掉volatile，子线程可能永远看不到flag的变化，程序不会停止。加上volatile，子线程能立刻感知到变化，正常退出。

## 四、JMM的happens-before原则

### 是什么

happens-before是JMM定义的一套规则，用来判断两个操作的先后顺序——如果A happens-before B，那么A的结果对B一定可见。

不用死记硬背所有规则，记住几个常用的就行：

- **程序次序规则**：同一个线程里，前面的代码 happens-before 后面的代码
- **volatile规则**：写volatile变量 happens-before 读这个volatile变量
- **锁规则**：解锁 happens-before 加锁
- **传递性**：A happens-before B，B happens-before C → A happens-before C

### 有什么用

给程序员一个明确的保证——不用猜"这两个操作谁先谁后"，按规则来就行。比如你在写volatile变量之前做的所有修改，其他线程读到这个volatile变量时都能看到。

## 五、为什么需要JMM

### 问题1：CPU缓存导致可见性问题

现代CPU都有多级缓存，每个核心有自己的缓存。线程在不同核心上运行时，改的是各自缓存里的数据，不会立刻同步到主内存，其他线程看不到。

JMM的解决方案：定义了volatile、synchronized等机制，保证在需要的时候数据能及时同步。

### 问题2：指令重排序导致有序性问题

编译器和CPU为了提高性能，会把指令重新排序。单线程没问题，多线程就可能出bug。

JMM的解决方案：volatile、synchronized、final等关键字都有禁止重排序的效果，保证多线程下的正确性。

### 问题3：不同平台内存模型不一样

不同CPU架构（x86、ARM）的内存模型不一样，在x86上跑没问题的代码，到ARM上可能就出bug。

JMM的解决方案：Java在硬件层面之上又封装了一层自己的内存模型，**屏蔽硬件差异**，做到"一次编写，到处运行"。

## 六、volatile的经典应用

### 场景1：状态标记位

最常用的场景——用volatile变量作为线程运行/停止的标记，比如上面的flag示例。

### 场景2：双重检查锁（DCL）单例

单例模式的双重检查锁写法中，instance变量必须加volatile，否则指令重排序可能导致其他线程拿到一个还没初始化完成的对象。

```java
public class Singleton {
    // 必须加volatile，禁止指令重排序
    private static volatile Singleton instance;

    private Singleton() {}

    public static Singleton getInstance() {
        if (instance == null) {          // 第一次检查（无锁）
            synchronized (Singleton.class) {
                if (instance == null) {  // 第二次检查（加锁）
                    instance = new Singleton();
                }
            }
        }
        return instance;
    }
}
```

为什么需要volatile？因为 `new Singleton()` 不是原子操作，可能被重排序。volatile保证对象初始化完成后才会把引用写出去。

## 知识点总结

1. JMM是Java内存模型，解决多线程下可见性、有序性、原子性三大问题。

2. 主内存=大白板（共享），工作内存=小笔记本（线程私有）。

3. volatile保证可见性和有序性，但不保证原子性。

4. volatile就像大喇叭，一个人改了立刻通知所有人。

5. happens-before规则是判断可见性的依据，不用猜，按规则来。

6. volatile经典用法：状态标记位、DCL单例模式。

7. 原子性问题要用原子类或synchronized解决。
