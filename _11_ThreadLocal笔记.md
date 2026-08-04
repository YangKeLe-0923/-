# ThreadLocal核心知识速览

## 一、什么是ThreadLocal

**定义** **ThreadLocal = Thread Local Variable（线程局部变量）**，用来创建线程私有的变量，每个线程有自己的独立副本，互不干扰。

它本质上就是每个员工的**"私人储物柜"**——每个人有自己的柜子，自己放自己拿，不用和别人抢，也不用担心别人动你的东西。

打个比方：ThreadLocal就像公司的储物柜

- 每个员工（线程）有自己独立的储物柜格子
- 你把自己的东西（数据）放进去，只有你自己能取到
- 别人的柜子里有什么你不知道，你的柜子里有什么别人也不知道
- 天然线程安全，不需要加锁

## 二、核心方法

- `set(value)` —— 给当前线程设置值
- `get()` —— 获取当前线程存的值，没有返回null
- `remove()` —— 删除当前线程的值（**必须调用，防止内存泄漏**）
- `withInitial(supplier)` —— 静态方法，设置初始值（JDK8+）

### 代码示例：线程隔离

```java
public class ThreadLocalDemo {
    // 每个线程有独立副本，初始值0
    private static ThreadLocal<Integer> count = ThreadLocal.withInitial(() -> 0);

    public static void main(String[] args) {
        for (int i = 0; i < 3; i++) {
            new Thread(() -> {
                int val = count.get() + 1;
                count.set(val);
                System.out.println(Thread.currentThread().getName()
                    + " 的值：" + val);  // 每个线程都是1，互不影响
                count.remove(); // 用完必须删！
            }).start();
        }
    }
}
```

3个线程都对count做+1，但结果每个线程都是1，而不是3——因为每个线程有自己的副本，互不干扰。

## 三、实现原理

### 数据存在哪

很多人以为数据存在ThreadLocal对象里，其实不是。**数据存在Thread对象里**。

每个Thread对象都有一个 `threadLocals` 变量，类型是 `ThreadLocal.ThreadLocalMap`。ThreadLocalMap是一个自定义的Map，key是ThreadLocal实例，value是存的值。

结构大概是这样：

```
Thread thread = ...;
thread.threadLocals = new ThreadLocalMap();
// ThreadLocalMap里的Entry：
// key: ThreadLocal实例（弱引用）
// value: 真正存的值（强引用）
```

### get() 方法执行流程

获取当前线程 Thread.currentThread()
拿到线程的 threadLocals（ThreadLocalMap）
以当前 ThreadLocal 对象为 key，从 map 中取值
返回 value

这就是为什么"ThreadLocal是线程隔离的"——每个线程有自己的Map，各存各的，互不影响。

### 弱引用key

ThreadLocalMap中的Entry的key（ThreadLocal实例）是**弱引用**，value是强引用。

弱引用的特点：GC的时候，如果没有强引用指向它，就会被回收。

设计原因：ThreadLocal对象用完了，方法执行完了，栈上的引用没了，如果key是强引用，那ThreadLocal对象永远回收不了。用弱引用，ThreadLocal对象就能正常被GC回收。

## 四、内存泄漏问题

### 为什么会内存泄漏

key是弱引用，GC后key变成null了，但value还是强引用，Thread还活着（线程池场景下线程不会死），ThreadMap也在，value就一直被引用着，回收不了。

这些key为null的Entry，value永远访问不到，但一直占着内存，时间长了就OOM了。

引用链：Thread → ThreadLocalMap → Entry（key=null, value强引用）→ value对象

### 什么场景最严重

**线程池 + ThreadLocal**是最容易出问题的组合。

因为线程池里的线程是复用的，用完了不会销毁，一直活着。如果ThreadLocal用完了不remove，value就一直挂在这个线程的ThreadLocalMap上，线程不死value就不回收。

### 怎么解决

**用完一定要调用 remove()**！这是最根本的解决方案。

最佳实践：try里用，finally里remove，保证一定执行到。

```
try {
    threadLocal.set(someValue);
    // 业务逻辑...
} finally {
    threadLocal.remove();  // 必须在finally里删
}
```

ThreadLocalMap自己也有一些清理机制（set/get/remove的时候会顺便清理一些key为null的Entry），但这不靠谱，不能依赖。

## 五、常见应用场景

### 场景1：用户上下文传递

Web应用中，把当前登录用户信息存在ThreadLocal里，在当前请求的整个调用链中随时可以取到，不用层层传递参数。

```java
public class UserContext {
    private static ThreadLocal<User> userHolder = new ThreadLocal<>();

    public static void setUser(User user) {
        userHolder.set(user);
    }
    public static User getUser() {
        return userHolder.get();
    }
    public static void clear() {
        userHolder.remove();
    }
}
```

拦截器里set，Controller/Service里get，拦截器的afterCompletion里remove。

### 场景2：数据库连接管理

保证同一个线程用同一个数据库连接，事务操作时同一个连接才能保证事务一致性。

### 场景3：SimpleDateFormat线程安全

SimpleDateFormat不是线程安全的，多线程共享会出问题。可以每个线程存一个自己的SimpleDateFormat。

不过JDK8之后推荐用DateTimeFormatter，它是线程安全的，不需要ThreadLocal。

## 六、为什么需要ThreadLocal

### 问题1：共享变量线程安全

多线程共享一个变量，需要加锁保证安全，加锁影响性能，还容易死锁。

ThreadLocal的解决方案：不给共享了，每人一份，天然线程安全，不用加锁，性能好。

### 问题2：参数层层传递太麻烦

用户信息、请求上下文这些数据，从Controller到Service到Dao，每层方法都要传参数，方法签名又长又丑。

ThreadLocal的解决方案：存在当前线程里，随时取，不用传参，**隐式传参**。

问题3：线程安全的工具类
有些工具类（如SimpleDateFormat）不是线程安全的，要么每次new一个（浪费性能），要么加锁（影响性能）。

ThreadLocal的解决方案：每个线程一个，既线程安全，又不用频繁创建对象。

## 知识点总结

1. ThreadLocal=员工储物柜，每个线程一份，互不干扰，天然线程安全。

2. 四个核心方法：set、get、remove、withInitial。

3. 数据存在Thread对象的ThreadLocalMap里，不是存在ThreadLocal里。

4. ThreadLocalMap的key是弱引用，value是强引用。

5. 内存泄漏：key被GC回收了（变null），value还被Thread强引用，回收不了。

6. 根本解法：用完在finally里调用remove()，线程池场景尤其要注意。

7. 典型应用：用户上下文、数据库连接、非线程安全工具类。

8. ThreadLocal是线程隔离，不是线程共享，别搞反了。
