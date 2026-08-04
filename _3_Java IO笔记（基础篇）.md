# Java IO核心知识速览（基础篇）

## 一、什么是 Java IO 流

**定义** **IO = Input/Output（输入/输出）**，Java中用来处理数据在不同设备之间传输的一套API。

它本质上就是一套**"数据管道系统"**——数据像水流一样，从一个地方通过管道流到另一个地方。输入是数据流入程序，输出是数据流出程序。

打个比方：Java IO就像城市的自来水管道系统

- 数据源（文件、网络）是水库，存储着原始数据
- 输入流是进水管，把水（数据）从水库抽到你家（程序内存）
- 输出流是出水管，把水（数据）从你家排出去（写入文件/网络）
- 缓冲流是水塔，攒满了再送，减少往返次数，提高效率

## 二、流的核心分类

### 分类1：按传输单位分——字节流 vs 字符流

**字节流**：以字节（byte）为单位传输，一次搬运1个字节。可以处理**所有类型文件**（图片、视频、音频、文本）。

**字符流**：以字符（char）为单位传输，一次搬运1个字符。只能处理**文本文件**，自动处理编码转换，避免中文乱码。

判断标准：能用记事本打开还能看懂的用字符流，看不懂的用字节流。

### 分类2：按流向分——输入流 vs 输出流

**输入流**：数据从外部（文件/网络）流向程序内存，用 read() 方法读数据。

**输出流**：数据从程序内存流向外部（文件/网络），用 write() 方法写数据。

判断标准：以程序为中心——进程序的是输入，出程序的是输出。

### 四大抽象基类

Java IO的所有流都继承自这4个抽象类：

- `InputStream` —— 字节输入流基类，核心方法 read()
- `OutputStream` —— 字节输出流基类，核心方法 write()
- `Reader` —— 字符输入流基类，核心方法 read()
- `Writer` —— 字符输出流基类，核心方法 write()

## 三、字节流详解

### 第 1 步：用 FileInputStream 读文件

文件字节输入流，从文件中读取字节数据。核心方法：

- `read()` —— 读一个字节，返回-1表示读完
- `read(byte[])` —— 读一批到字节数组，返回读到的字节数
- `close()` —— 关闭流，释放资源

```java
// 推荐写法：用字节数组一次读一批（效率高）
try (FileInputStream fis = new FileInputStream("a.txt")) {
    byte[] buf = new byte[1024];  // 1KB缓冲区
    int len;
    while ((len = fis.read(buf)) != -1) {  // 一次读1KB
        System.out.print(new String(buf, 0, len));
    }
} catch (IOException e) {
    e.printStackTrace();
}
```

**注意**：用 try-with-resources 自动关流，不用手动写 finally。这是JDK7后的推荐写法。

### 第 2 步：用 FileOutputStream 写文件

文件字节输出流，把字节数据写入文件。核心方法：

- `write(int)` —— 写一个字节
- `write(byte[])` —— 写一个字节数组
- `write(byte[], off, len)` —— 写字节数组的一部分

```
// 文件复制：边读边写
try (FileInputStream fis = new FileInputStream("src.jpg");
     FileOutputStream fos = new FileOutputStream("dest.jpg")) {
    byte[] buf = new byte[8192];  // 8KB缓冲
    int len;
    while ((len = fis.read(buf)) != -1) {
        fos.write(buf, 0, len);   // 读多少写多少
    }
} catch (IOException e) {
    e.printStackTrace();
}
```

复制文件是字节流最经典的用法——图片、视频等二进制文件只能用字节流复制，用字符流会损坏文件。

## 四、字符流详解

### 为什么需要字符流

字节流处理文本有个致命问题：**中文乱码**。UTF-8编码下一个中文占3个字节，如果用字节流逐字节读取，会把一个汉字拆成3半，强转成char就变成乱码了。

字符流 = 字节流 + 编码表。它内部自动按编码把多个字节组合成一个完整字符，开发者不用操心编码问题。

### 第 1 步：用 FileReader 读文本

文件字符输入流，默认使用系统编码（Windows是GBK）。核心方法和字节流类似，但操作单位是char：

```java
try (FileReader fr = new FileReader("日记.txt")) {
    char[] cbuf = new char[1024];
    int len;
    while ((len = fr.read(cbuf)) != -1) {
        System.out.print(new String(cbuf, 0, len));
    }
} catch (IOException e) {
    e.printStackTrace();
}
```

**坑**：FileReader不能指定编码，文件编码和系统编码不一致时会乱码。解决方案：用 InputStreamReader 指定编码。

### 第 2 步：用 FileWriter 写文本

文件字符输出流，写字符数据到文件。有两个特殊方法：

- `write(String)` —— 直接写字符串，不用转成字符数组
- `flush()` —— 刷新缓冲区，把内存中缓存的数据刷入磁盘

```
try (FileWriter fw = new FileWriter("笔记.txt")) {
    fw.write("今天学习了Java IO流\n");  // 直接写字符串
    fw.write("字节流处理二进制，字符流处理文本");
    fw.flush();  // 刷入磁盘
} catch (IOException e) {
    e.printStackTrace();
}
```

**flush vs close**：flush只刷新缓冲区，流还能用；close先flush再关闭流，流不能再用了。关流前会自动flush，所以一般不用手动调。

## 五、为什么要有两种流

### 问题 1：只用字节流行不行

行，但处理文本很麻烦。你需要手动处理编码，判断一个字符占几个字节，一不小心就乱码。

字符流就是为了让你**更方便地处理文本**——底层还是字节流，只是帮你把编码转换的活干了。

### 问题 2：只用字符流行不行

不行。图片、视频、音频等二进制文件没有"字符"的概念，用字符流处理会破坏文件结构，导致文件损坏。

**字节流是万能的，字符流是专门处理文本的**——底层统一，上层按需增强。

## 六、流选择指南

### 决策步骤

- 第一步：是文本文件吗？是 → 字符流；否 → 字节流
- 第二步：是读还是写？读 → InputStream/Reader；写 → OutputStream/Writer
- 第三步：需要提高效率吗？需要 → 包一层 BufferedXxx 缓冲流
- 第四步：需要指定编码吗？需要 → 用 InputStreamReader/OutputStreamWriter

### 常见场景速查

- 复制图片/视频 → FileInputStream + FileOutputStream + 8KB数组
- 读取文本文件 → BufferedReader + FileReader
- 写入文本文件 → BufferedWriter + FileWriter
- 读取指定编码的文本 → BufferedReader + InputStreamReader（指定UTF-8）

## 知识点总结

1. IO流就像管道系统，输入流进水，输出流出水。

2. 字节流处理一切文件，字符流只处理文本，但自动处理编码。

3. 四大基类：InputStream、OutputStream、Reader、Writer。

4. 字节流复制文件经典写法：8KB数组 + while循环 + try-with-resources。

5. 字符流 = 字节流 + 编码表，本质还是字节流。

6. flush只刷新不关闭，close先刷新再关闭。

7. 选流口诀：文本用字符，二进制用字节；需要效率加缓冲，指定编码用转换流。
