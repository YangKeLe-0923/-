# ForkJoin框架笔记

## 一、什么是Fork/Join框架

**定义** **Fork/Join** 是Java并发包提供的并行计算框架，专门用于支持**分治任务模型**——把大任务拆成小任务，小任务算完了再合并结果。

Fork就是"分叉/拆分"，Join就是"合并/汇合"。可以理解为**单机版的MapReduce**。

打个比方：就像**公司做年终总结报告**

- 老板要一份全公司的年终总结（大任务）
- 总经理把任务拆给各个部门经理（Fork拆分）
- 部门经理再拆给各个小组长，小组长拆给员工
- 员工写完自己的部分，交给组长汇总（Join合并）
- 组长汇总完交给经理，经理汇总完交给总经理，最后合并成完整报告
- 层层拆分、并行干活、层层合并——这就是分治思想

## 二、核心组件与API

### ForkJoinPool：分治线程池

这是Fork/Join的核心，是执行分治任务的线程池。和普通线程池不同，它有"工作窃取"机制。

```
ForkJoinPool pool = new ForkJoinPool(4);  // 4个工作线程
```

### ForkJoinTask：分治任务

分治任务的基类，有两个常用子类：

- **RecursiveTask**：有返回值的分治任务，compute方法返回结果
- **RecursiveAction**：无返回值的分治任务，compute方法是void

核心方法：

```
task.fork();    // 异步执行一个子任务（把任务压入当前线程的队列）
task.join();    // 阻塞等待子任务执行结果
pool.invoke(task);  // 提交任务并等待返回结果
task.compute(); // 子类实现：定义拆分逻辑 + 最小任务计算逻辑
```

### 完整示例：斐波那契数列

```java
class Fibonacci extends RecursiveTask<Integer> {
    final int n;
    Fibonacci(int n) { this.n = n; }

    @Override
    protected Integer compute() {
        // 最小可计算单元：n<=1直接返回
        if (n <= 1) { return n; }

        // 拆分成两个子任务
        Fibonacci f1 = new Fibonacci(n - 1);
        f1.fork();  // f1异步执行

        Fibonacci f2 = new Fibonacci(n - 2);
        int f2Result = f2.compute();  // f2在当前线程计算
        int f1Result = f1.join();     // 等待f1的结果

        return f1Result + f2Result;   // 合并结果
    }
}

// 运行
ForkJoinPool pool = new ForkJoinPool(4);
Integer result = pool.invoke(new Fibonacci(20));
System.out.println(result);  // 6765
```

标准写法：一个子任务fork出去异步执行，另一个在当前线程直接compute（节省一次fork开销），最后join等待fork的那个。

## 三、工作窃取机制

**定义** **工作窃取（Work Stealing）** 是ForkJoinPool的核心特色：当某个工作线程自己的任务队列空了，它会去"偷"其他线程队列里的任务来执行。

目的是让所有线程都保持忙碌，不浪费CPU资源。

打个比方：就像**快递员送快递**

- 每个快递员都有自己的一车快递（任务队列），正常情况下先送自己的
- 有的快递员送得快，自己的快递送完了（队列为空）
- 他不会闲着，而是去帮送得慢的同事送——从同事的车后面拿一件来送（窃取）
- 这样大家都不停下来，整体效率更高

### 双端队列设计

任务队列是双端队列（Deque），这是工作窃取的关键：

- 正常情况下，线程从队列的**头部**取自己的任务（LIFO，后进先出）
- 窃取时，线程从其他队列的**尾部**偷任务
- 头尾各取各的，减少竞争——就像你从前面拿，我从后面偷，尽量不撞上

### 与普通线程池的区别

对比项ThreadPoolExecutorForkJoinPool
任务队列一个公共队列（FIFO）每个线程一个双端队列（LIFO）
核心机制线程复用工作窃取
适用场景普通并行任务分治任务（有父子依赖）
子任务结果用Future.get获取用join直接获取，更方便

## 四、使用场景

### 场景1：大数据量计算

比如计算1到1亿的和，可以拆成1000个子任务，每个算10万个数，最后加起来。

CPU密集型任务用ForkJoin效果最好，能充分利用多核CPU。

### 场景2：分治算法

归并排序、快速排序、二分查找这些经典分治算法，都能用ForkJoin并行化。

### 场景3：Java 8并行流

Java 8的Stream并行流（`parallelStream()`）底层就是用ForkJoinPool实现的。默认共享一个公共ForkJoinPool，线程数等于CPU核数。

```
// 底层走的就是ForkJoinPool.commonPool()
long count = list.parallelStream()
                 .filter(x -> x > 100)
                 .count();
```

## 五、关键技巧与坑点

### 技巧1：合理设置任务拆分粒度

任务不是拆得越细越好，拆得太细，拆分和合并的开销比计算本身还大，反而变慢。

一般经验：最小任务的计算量应该在几千到几万次CPU操作之间，根据实际情况调整阈值。

### 技巧2：一个fork一个compute的写法

拆成两个子任务时，不要两个都fork再都join——这样当前线程就闲着了。

标准写法是：一个fork出去，另一个在当前线程直接compute，然后join等fork的那个。当前线程不浪费，这是一个常见的优化点。

### 坑点1：默认并行流共享线程池

**现象**：系统里多个地方用了parallelStream，有时候一个地方的慢任务会拖慢其他地方的并行流。

**原因**：默认所有并行流都共享 `ForkJoinPool.commonPool()`，线程数默认是CPU核数-1。如果有IO密集型任务占着线程不放，其他任务就得等。

**解法**：IO密集型的并行任务，自己new一个ForkJoinPool来跑，不要用公共池。CPU密集型用公共池没问题。

### 坑点2：不是所有任务都适合ForkJoin

**现象**：用了ForkJoin反而比普通for循环还慢。

**原因**：数据量太小，或者任务不是CPU密集型的，拆分和线程调度的开销超过了并行带来的收益。

**解法**：小任务、IO密集型任务就别用了。ForkJoin适合大计算量的CPU密集型分治任务。

### 坑点3：任务不能太多太小

**现象**：任务拆得特别细，结果内存占用飙升，GC频繁。

**原因**：每个子任务都是一个对象，拆成几百万个小任务，光任务对象就占很多内存。

**解法**：合理设置阈值，不要拆太细。一般来说任务数控制在CPU核数的几倍到几十倍就够了。

### 坑点4：join会阻塞当前线程

**现象**：在普通线程池里调用ForkJoinTask的join，线程阻塞了，影响其他任务。

**原因**：join是阻塞等待，在ForkJoinPool里调用有优化（会去做别的任务），但在外面调用就是真的阻塞。

**解法**：ForkJoinTask的join尽量在ForkJoinPool的工作线程里调用，能享受到工作窃取的优化。

## 六、总结

- Fork/Join = 分治任务框架，大任务拆小任务，小任务算完合并结果，单机版MapReduce
- 两个核心：ForkJoinPool（线程池）+ ForkJoinTask（任务）
- RecursiveTask有返回值，RecursiveAction无返回值
- 工作窃取是核心特色：线程干完自己的活会去偷别人的活，提高CPU利用率
- 双端队列设计：自己从头部取（LIFO），偷从尾部偷，减少竞争
- 标准写法：一个子任务fork，另一个直接compute，最后join，充分利用当前线程
- 适合CPU密集型分治任务，不适合小任务和IO密集型
- Java 8并行流底层就是ForkJoinPool，默认共享公共池，IO密集型任务建议自建池
