# 原子类与Unsafe笔记

## 一、什么是原子操作类

**定义** **原子操作类 = java.util.concurrent.atomic包**，是Java提供的一系列支持**无锁原子操作**的工具类。

它能让你在多线程环境下，对单个变量进行线程安全的读写，而不需要用synchronized加锁。

打个比方：原子类就像**自动售货机**

- 普通变量（如int）就像开放式货架，两个人同时拿东西会抢——线程不安全
- synchronized就像派个保安守着货架，一次只让一个人拿——安全但慢
- 原子类就像自动售货机，你投币、选货、出货一气呵成，别人插不进来——安全又快

最常见的场景是计数器：`count++` 看起来是一行代码，实际分"读→加1→写回"三步，多线程下会丢数据。用 `AtomicInteger` 就能一步完成，不会出错。

## 二、原子类四大分类

### 1. 基本数据类型

对基本类型变量做原子操作，最常用的一类。

```
AtomicInteger  // 原子int
AtomicLong     // 原子long
AtomicBoolean  // 原子boolean（底层用int实现，true=1，false=0）
```

核心方法（以AtomicInteger为例）：

```
atomicInteger.incrementAndGet();    // 加1，返回新值
atomicInteger.getAndIncrement();    // 加1，返回旧值
atomicInteger.addAndGet(5);         // 加5，返回新值
atomicInteger.getAndSet(100);       // 设置新值，返回旧值
atomicInteger.compareAndSet(5, 10); // 期望是5就改成10，成功返回true
```

### 2. 数组类型

对数组中的某个元素做原子操作，不是整个数组。

```
AtomicIntegerArray   // 原子int数组
AtomicLongArray      // 原子long数组
AtomicReferenceArray // 原子引用数组
```

注意：构造时会拷贝一份数组，修改原子数组不会影响原始数组。

### 3. 引用类型

对对象引用做原子替换，还支持带版本号的引用解决ABA问题。

```
AtomicReference           // 原子引用替换
AtomicStampedReference    // 带版本号的引用，解决ABA问题
AtomicMarkableReference   // 带标记的引用（boolean标记）
```

### 4. 字段更新类型

只想原子更新对象的某个字段，而不是整个对象。要求字段必须是 `public volatile`。

```
AtomicIntegerFieldUpdater  // 更新int字段
AtomicLongFieldUpdater     // 更新long字段
AtomicReferenceFieldUpdater // 更新引用字段
```

## 三、底层原理：CAS + Unsafe

### CAS机制

原子类的核心是 **CAS（Compare-And-Swap，比较并交换）**，一种无锁技术。

CAS包含三个操作数：内存位置V、预期原值A、新值B。当且仅当V等于A时，才原子地把V更新为B，否则什么都不做。

打个比方：CAS就像**超市换货**

- 你买了瓶可乐（A），想换成雪碧（B）
- 你拿着可乐去柜台，只有货架上还是可乐（V==A）时，才能换成雪碧
- 如果货架上已经不是可乐了（被别人换走了），就不给你换——操作失败
- 整个过程由CPU原子指令保证，不会被打断

### Unsafe类是基石

所有原子类的CAS操作，底层都调用 `sun.misc.Unsafe` 类的native方法。Unsafe是Java的"后门"，能直接操作内存和线程。

AtomicInteger的getAndIncrement源码逻辑：

```
public final int getAndIncrement() {
    // this: AtomicInteger实例
    // valueOffset: value字段的内存偏移量
    // 1: 要增加的值
    // 返回增加前的原始值
    return unsafe.getAndAddInt(this, valueOffset, 1);
}
```

`valueOffset` 表示value字段在对象内存中的偏移量，Unsafe通过这个偏移量直接定位到内存地址，执行CAS操作。

## 四、什么是Unsafe

**定义** **Unsafe = sun.misc.Unsafe**，是Java中一个非常特殊的类，提供了底层、"不安全"的机制来直接访问和操作内存、线程和对象。

它被final修饰，不能继承，构造方法是private的，以单例模式存在。只有启动类加载器加载的类才能直接调用，普通类调用会抛SecurityException。

打个比方：Unsafe就像**游戏里的外挂/控制台**

- 正常玩游戏（普通Java代码）只能用游戏提供的操作，不能穿墙不能飞天
- 开了外挂（Unsafe）可以直接修改内存数值，血量、金币随便改
- 用好了很爽，用不好游戏直接崩溃——JVM直接崩给你看

## 五、Unsafe核心能力

### 1. 获取Unsafe实例

不能直接 `Unsafe.getUnsafe()`，会检查类加载器。需要通过反射绕过：

```
public static Unsafe getUnsafe() throws Exception {
    Field unsafeField = Unsafe.class.getDeclaredField("theUnsafe");
    unsafeField.setAccessible(true);
    return (Unsafe) unsafeField.get(null);
}
```

### 2. 内存操作

直接读写对象的内存，可以绕过访问权限修改private字段。

```
// 获取字段的内存偏移量
long offset = unsafe.objectFieldOffset(User.class.getDeclaredField("age"));

// 直接修改private字段的值，不用setter
unsafe.putInt(user, offset, 99);
```

### 3. CAS原子操作

这是juc包所有并发工具的基石，AQS、原子类、ConcurrentHashMap都靠它。

```
// 原子比较并交换int值
// 对象o的offset偏移处，如果当前值是expected，就改成x
boolean success = unsafe.compareAndSwapInt(o, offset, expected, x);
```

### 4. 内存屏障

手动插入内存屏障，防止指令重排序，实现volatile的效果。

```
unsafe.loadFence();   // 读屏障：禁止load操作重排序
unsafe.storeFence();  // 写屏障：禁止store操作重排序
unsafe.fullFence();   // 全屏障：禁止所有重排序
```

### 5. 绕过构造器创建对象

`allocateInstance` 可以不调用构造方法直接创建对象实例。

```
// 不会调用构造方法，成员变量都是默认值（0、null、false）
User user = (User) unsafe.allocateInstance(User.class);
```

## 六、关键技巧与坑点

### 技巧1：用AtomicStampedReference解决ABA问题

**ABA问题**：变量从A变成B又变回A，普通CAS会误认为"没变过"。比如你拿可乐去换雪碧，中间有人把可乐换成了水又换回可乐，CAS检查还是可乐就同意换了，但中间已经发生过变化。

**解法**：用 `AtomicStampedReference`，每次修改都带版本号，不仅比较值还比较版本号。

### 技巧2：字段更新器节省内存

如果一个类有很多对象，每个对象都用AtomicInteger存一个字段，内存开销大。用AtomicIntegerFieldUpdater可以只创建一个更新器对象，所有实例共用，节省内存。

前提是字段必须声明为 `public volatile`。

### 坑点1：原子类只能保证单个变量原子性

**现象**：用了AtomicInteger，两个变量的操作还是有并发问题。

**原因**：原子类只能保证单个变量的单次操作是原子的，多个变量之间的复合操作不保证。

**解法**：多个变量需要原子操作时，用AtomicReference把它们封装成一个对象，或者直接用锁。

### 坑点2：Unsafe不受官方支持，版本不稳定

**现象**：在JDK8能用的Unsafe方法，JDK11以后用不了了。

**原因**：Unsafe是JDK内部API，不属于Java标准，Java 9后被逐步废弃和迁移。

**解法**：业务代码尽量不要直接用Unsafe，优先用juc包封装好的工具类。

### 坑点3：堆外内存泄漏

**现象**：用Unsafe分配堆外内存，程序运行久了内存溢出。

**原因**：堆外内存不受GC管理，分配了不手动释放就会一直占用。

**解法**：用try-finally确保 `freeMemory` 一定会被调用。

## 七、总结

- 原子类提供无锁的线程安全方案，底层是CAS机制，比synchronized轻量
- 原子类分四大类：基本类型、数组、引用、字段更新器
- CAS = 比较并交换，只有期望值匹配时才更新，由CPU原子指令保证
- Unsafe是原子类和整个juc包的底层基石，提供内存操作、CAS、内存屏障等能力
- Unsafe不能直接获取，需通过反射，属于JDK内部API，不建议业务代码直接用
- ABA问题用AtomicStampedReference（带版本号）解决
- 原子类只保证单个变量原子性，多变量复合操作需用锁或AtomicReference封装
- Unsafe分配的堆外内存不受GC管理，必须手动释放防止内存泄漏
