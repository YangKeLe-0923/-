# Java IO核心知识速览（进阶篇）

## 一、缓冲流

**定义** **缓冲流 = BufferedXxx**，是在基础流外面包了一层缓冲区的装饰流，用来大幅提升IO效率。

它本质上就是给基础流加了一个**"水桶"**——不是一滴水一滴水地搬，而是用桶装满了再搬，减少往返次数。

打个比方：缓冲流就像快递驿站的分拣仓库

- 普通流是"上门取件"，每个快递跑一趟，效率极低
- 缓冲流是"攒满一车再送"，把快递集中起来一次性拉走
- 默认缓冲区大小是 8KB（8192字节），相当于一车的容量

### 四类缓冲流

- `BufferedInputStream` —— 字节输入缓冲流，包在 InputStream 外面
- `BufferedOutputStream` —— 字节输出缓冲流，包在 OutputStream 外面
- `BufferedReader` —— 字符输入缓冲流，多了 readLine() 方法按行读
- `BufferedWriter` —— 字符输出缓冲流，多了 newLine() 方法写换行

### 代码示例：缓冲流复制文件

```
// 缓冲流 + 字节数组，效率最高的组合
try (BufferedInputStream bis = new BufferedInputStream(new FileInputStream("src.mp4"));
     BufferedOutputStream bos = new BufferedOutputStream(new FileOutputStream("dest.mp4"))) {
    byte[] buf = new byte[8192];  // 8KB数组
    int len;
    while ((len = bis.read(buf)) != -1) {
        bos.write(buf, 0, len);
    }
} catch (IOException e) {
    e.printStackTrace();
}
```

**性能对比**（复制370MB文件）：普通流逐字节10分钟以上，普通流+8KB数组约8秒，缓冲流+8KB数组约0.5秒。差距的核心在于**磁盘IO次数**。

## 二、转换流

**定义** **转换流 = InputStreamReader / OutputStreamWriter**，是字节流和字符流之间的桥梁，可以指定编码格式。

它本质上就是一个**"翻译官"**——字节流说的是机器语言（0和1），字符流说的是人类语言（文字），转换流在中间当翻译，按指定的编码规则互转。

### 核心作用

- 字节流转字符流 —— InputStreamReader：读的时候指定编码
- 字符流转字节流 —— OutputStreamWriter：写的时候指定编码
- 解决中文乱码的终极武器 —— 手动指定UTF-8/GBK

### 代码示例：读取GBK编码的文件

```java
// 指定GBK编码读取，不会乱码
try (BufferedReader br = new BufferedReader(
        new InputStreamReader(new FileInputStream("老文件.txt"), "GBK"))) {
    String line;
    while ((line = br.readLine()) != null) {  // 按行读
        System.out.println(line);
    }
} catch (IOException e) {
    e.printStackTrace();
}
```

**乱码原因**：编码和解码用的不是同一个编码表。存的时候用GBK，读的时候用UTF-8，就像用中文密码本去解密文写的信，肯定看不懂。解决方案：统一用UTF-8。

## 三、File 类

**定义** **File = 文件/目录路径的抽象表示**，用来操作文件和目录本身（创建、删除、重命名等），**不能读写文件内容**。

它本质上就是一个**"户口本"**——只登记文件的基本信息（叫什么、在哪、多大），不关心文件里面装了什么内容。

### 常用方法

- `exists()` —— 判断文件/目录是否存在
- `createNewFile()` —— 创建新文件
- `mkdir() / mkdirs()` —— 创建目录（mkdirs可以创建多级目录）
- `delete()` —— 删除文件/目录
- `length()` —— 获取文件大小（字节数）
- `listFiles()` —— 获取目录下所有子文件和子目录

### 代码示例：遍历目录

```java
// 递归遍历目录下所有文件
public static void listAll(File dir) {
    File[] files = dir.listFiles();
    if (files != null) {
        for (File f : files) {
            if (f.isDirectory()) {
                System.out.println("目录：" + f.getName());
                listAll(f);  // 递归子目录
            } else {
                System.out.println("文件：" + f.getName()
                    + "，大小：" + f.length() + "字节");
            }
        }
    }
}
```

## 四、IO异常处理

### JDK7 之前：手动关流（繁琐易错）

需要在 finally 里关闭流，还要判断 null，还要 try-catch 嵌套，代码非常臃肿：

```
FileInputStream fis = null;
try {
    fis = new FileInputStream("a.txt");
    // 读文件...
} catch (IOException e) {
    e.printStackTrace();
} finally {
    if (fis != null) {  // 先判断非空
        try {
            fis.close();  // 关流还要再try-catch
        } catch (IOException e) {
            e.printStackTrace();
        }
    }
}
```

### JDK7 之后：try-with-resources（推荐）

只要资源实现了 AutoCloseable 接口，放在 try 后面的括号里，**用完自动关闭**，不用写finally：

```
// 自动关流，代码简洁，不会忘记关
try (FileInputStream fis = new FileInputStream("a.txt");
     FileOutputStream fos = new FileOutputStream("b.txt")) {
    byte[] buf = new byte[8192];
    int len;
    while ((len = fis.read(buf)) != -1) {
        fos.write(buf, 0, len);
    }
} catch (IOException e) {
    e.printStackTrace();
}
```

**注意**：所有IO流都实现了 AutoCloseable 接口，都可以用 try-with-resources。多个资源用分号隔开，会自动按倒序关闭。

## 五、为什么需要这些增强流

### 问题 1：普通流效率太低

普通字节流一次读写一个字节，每次都要访问磁盘。磁盘IO比内存慢几千倍，频繁磁盘IO就是性能瓶颈。

缓冲流的解决方案：用8KB内存缓冲区，一次读满再处理，磁盘IO次数从数百万次降到几万次，**用内存空间换磁盘时间**。

### 问题 2：编码问题太头疼

FileReader/FileWriter默认用系统编码，跨平台就乱码。手动处理编码又很麻烦。

转换流的解决方案：在字节流和字符流之间架一座桥，显式指定编码格式，**从根源上解决乱码问题**。

### 问题 3：流关不好会泄漏

IO资源用完必须关闭，否则文件句柄泄漏，时间长了系统会报错"打开的文件太多"。手动写finally又容易漏。

try-with-resources的解决方案：编译器自动生成关闭代码，**不会忘，不会错**。

## 六、装饰者模式

### 设计思想

Java IO流的类体系是**装饰者模式**的经典应用。所有流都实现相同的接口，可以层层包装，每层增加新能力：

- 最内层：FileInputStream —— 基础文件流，连接数据源
- 包一层：BufferedInputStream —— 加缓冲，提高效率
- 再包一层：DataInputStream —— 增加读写基本类型的能力

好处：功能可以**自由组合**，需要什么能力就包什么层，不需要修改原有类。符合开闭原则。

### 经典组合写法（从内到外）

```
// 字符缓冲流 = 缓冲 + 编码转换 + 文件
BufferedReader br = new BufferedReader(          // 第3层：缓冲+按行读
    new InputStreamReader(                       // 第2层：字节转字符
        new FileInputStream("a.txt"), "UTF-8")); // 第1层：文件字节流
```

三层各司其职：最内层打开文件，中间层做编码转换，最外层加缓冲。调用 readLine() 的时候，数据从磁盘经过三层处理到达你的代码。

## 知识点总结

1. 缓冲流=给水桶，8KB缓冲区，减少磁盘IO，大幅提升效率。

2. 转换流=翻译官，字节转字符可指定编码，解决中文乱码。

3. File类只操作文件本身（创建删除重命名），不能读写内容。

4. try-with-resources自动关流，推荐写法，替代finally。

5. Java IO是装饰者模式的经典实现，层层包装，功能自由组合。

6. 经典组合：BufferedReader + InputStreamReader + FileInputStream。

7. 乱码本质：编码和解码用的不是同一张编码表，统一UTF-8就好。
