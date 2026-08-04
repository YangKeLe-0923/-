# ConcurrentHashMap核心知识速览

## 一、什么是ConcurrentHashMap

**定义** **ConcurrentHashMap** 是Java并发包（J.U.C）中的线程安全HashMap，支持高并发读写，性能比Hashtable和Collections.synchronizedMap好很多。

它本质上就是一个**"分段包间餐厅"**——不是整个餐厅只有一把锁，而是分成多个包间，每个包间有自己的锁。不同包间的客人互不影响，可以同时用餐。

打个比方：ConcurrentHashMap就像有多个收银台的超市

- 普通HashMap=无收银台，大家都挤在一起，乱成一锅粥（线程不安全）
- Hashtable=只有一个收银台，所有人排队，效率极低（全表锁）
- ConcurrentHashMap=有16个收银台，每个收银台排自己的队，不同收银台不冲突（分段锁/CAS）

## 二、JDK7 vs JDK8 实现对比

### JDK7：分段锁（Segment）

内部由一个Segment数组组成，每个Segment是一个独立的小HashMap，有自己的锁。

默认16个Segment，理论上最多支持16个线程同时写（每个操作不同的Segment）。

缺点：分段还是粗了，同一个Segment内还是只有一个线程能写；扩容只能在Segment内扩，不能全表扩容。

### JDK8：CAS + synchronized

JDK8彻底改造了实现，和HashMap一样用数组+链表+红黑树结构。不再用分段锁，而是用**CAS + 数组元素级别的synchronized**。

- **写操作**：先看数组对应位置是不是null，如果是null就用CAS放元素，成功就完事了
- **哈希冲突**：如果位置上已经有元素了，就用synchronized锁住这个位置的头节点，然后操作链表/红黑树
- **读操作**：全程无锁，volatile保证可见性，性能极高

JDK8的锁粒度更细——锁的是数组的每个元素（每把锁只锁一个桶），而不是一整个Segment。并发度更高，性能更好。

## 三、核心特性

### 特性1：弱一致性迭代器

ConcurrentHashMap的迭代器（iterator）是弱一致性的，不是强一致性的——迭代过程中如果有其他线程修改了map，迭代器不会抛ConcurrentModificationException。

迭代器能看到迭代开始后部分修改，但不一定能看到所有修改。

**好处**：迭代的时候不用加锁，不影响并发性能。**坏处**：不能保证看到最新的数据。

### 特性2：不允许null键和null值

HashMap可以存null键和null值，ConcurrentHashMap不行，放null会抛NullPointerException。

原因：多线程环境下，get(key)返回null的时候，你分不清是key不存在，还是value本身就是null，容易产生歧义。

### 特性3：size() 不是精确值

ConcurrentHashMap的size()方法返回的是一个估算值，不一定是精确值。因为计算的时候可能有其他线程在增删元素。

JDK8用CounterCell数组来统计元素个数，类似LongAdder的思路，减少竞争。

**注意**：不要在并发场景下用size()做精确判断，比如"if (map.size() == 0)"这种判断不可靠。

## 四、常用方法

- `put(key, value)` —— 添加键值对
- `get(key)` —— 根据key取value，无锁，性能高
- `remove(key)` —— 删除键值对
- `containsKey(key)` —— 判断key是否存在
- `putIfAbsent(key, value)` —— key不存在才放（原子操作）
- `computeIfAbsent(key, mappingFunction)` —— key不存在就计算并放入（原子操作，避免重复计算）

### 代码示例：并发统计

```java
public class ConcurrentHashMapDemo {
    public static void main(String[] args) throws InterruptedException {
        ConcurrentHashMap<String, Integer> map = new ConcurrentHashMap<>();

        // 10个线程同时统计
        for (int i = 0; i < 10; i++) {
            new Thread(() -> {
                for (int j = 0; j < 1000; j++) {
                    // key不存在设为1，存在就加1（注意：不是原子的，这里只是示例）
                    String key = "count_" + (j % 100);
                    map.compute(key, (k, v) -> v == null ? 1 : v + 1);
                }
            }).start();
        }
        Thread.sleep(2000);
        System.out.println("总key数：" + map.size());
    }
}
```

**注意**：put+get这种组合操作不是原子的。如果需要"如果不存在就添加"这种复合操作，用putIfAbsent或compute方法，它们是原子的。

## 五、为什么需要ConcurrentHashMap

### 问题1：HashMap线程不安全

多线程下同时put可能导致数据丢失，JDK7下甚至可能出现死循环（链表成环，get的时候死循环CPU 100%）。

ConcurrentHashMap的解决方案：用CAS和synchronized保证线程安全，同时尽量降低锁的粒度。

### 问题2：Hashtable性能太差

Hashtable用synchronized修饰方法，相当于整表加锁。不管读写哪个元素，都要等同一把锁。并发高的时候大家都排队，性能极差。

ConcurrentHashMap的解决方案：JDK7用分段锁（多把锁），JDK8用CAS+桶级锁，锁粒度大大降低，并发性能提升数倍。

### 问题3：Collections.synchronizedMap也不行

Collections.synchronizedMap是包装了一层synchronized的HashMap，本质和Hashtable一样，也是全表锁，性能一样差。

## 六、常见坑点

### 坑1：put+get不是原子的

很多人以为ConcurrentHashMap里所有操作都是线程安全的，写什么都行。不对——单个方法是原子的，但方法组合不是。

比如"if (!map.containsKey(key)) { map.put(key, value); }" 这两行之间可能被其他线程插入，导致重复put。

**解法**：用putIfAbsent、computeIfAbsent等原子复合操作。

### 坑2：迭代的时候修改不会抛异常

用HashMap的时候迭代中修改会抛ConcurrentModificationException（fail-fast），很多人习惯了靠异常发现bug。

但ConcurrentHashMap不会抛——它是弱一致性的，迭代过程中允许修改。不要指望靠异常来发现并发修改问题。

### 坑3：size()不是精确值

不要用size() == 0来判断map是否为空，多线程下判断完了可能立刻就变了。

**解法**：用isEmpty()也一样，都是弱一致的。如果需要精确判断，得自己加锁或者用其他同步方式。

## 知识点总结

1. ConcurrentHashMap是线程安全的HashMap，性能远高于Hashtable。

2. JDK7分段锁（16个Segment），JDK8用CAS+桶级synchronized，锁粒度更细。

3. 读操作全程无锁，volatile保证可见性，读性能极高。

4. 迭代器是弱一致性的，不会抛ConcurrentModificationException。

5. 不允许null键和null值，避免歧义。

6. size()是估算值，不是精确值，不要用来做精确判断。

7. 单个方法是原子的，方法组合不是原子的，用putIfAbsent等原子方法。
