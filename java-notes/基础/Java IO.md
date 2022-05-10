## 一、IO概念

Java中I/O操作主要是指使用Java进行输入，输出操作。 Java所有的I/O机制都是基于数据流进行输入输出，这些数据流表示了字符或者字节数据的流动序列。Java的I/O流提供了读写数据的标准方法。任何Java中表示数据源的对象都会提供以数据流的方式读写它的数据的方法。 

在电脑上的数据有三种存储方式，一种是外存，一种是内存，一种是缓存。比如电脑上的硬盘，磁盘，U盘等都是外存，在电脑上有内存条，缓存是在CPU里面的。外存的存储量最大，其次是内存，最后是缓存，但是外存的数据的读取最慢，其次是内存，缓存最快。这里总结从外存读取数据到内存以及将数据从内存写到外存中。

标准输入输出，文件的操作，网络上的数据流，字符串流，对象流，zip文件流等等，Java中将输入输出抽象称为流，就好像水管，将两个容器连接起来。将数据从外存中读取到内存中的称为输入流，将数据从内存写入外存中的称为输出流。

## 二、IO分类

![IO7](D:\notes\java-notes\资源\IO7.png)

### 2.1、根据流向分为输入流和输出流

输出：把程序(内存)中的内容输出到磁盘、光盘等存储设备中

![IO1](D:\notes\java-notes\资源\IO1.png)

输入：读取外部数据（磁盘、光盘等存储设备的数据）到程序（内存）中

![IO2](D:\notes\java-notes\资源\IO2.png)

综合起来：

![IO3](D:\notes\java-notes\资源\IO3.png)

### 2.2、根据传输数据单位分为字节流和字符流

1. 字节流：数据流中最小的数据单元是字节 
2. 字符流：数据流中最小的数据单元是字符， Java中的字符是Unicode编码，一个字符占用两个字节

### 2.3、根据功能分为节点流和包装流

节点流：可以从或向一个特定的地方(节点)读写数据，直接连接数据源。如最常见的是文件的FileReader，还可以是数组、管道、字符串，关键字分别为ByteArray/CharArray，Piped，String。.

处理流（包装流）：并不直接连接数据源，是对一个已存在的流的连接和封装，是一种典型的装饰器设计模式，使用处理流主要是为了更方便的执行输入输出工作，如PrintStream，输出功能很强大，又如BufferedReader提供缓存机制，推荐输出时都使用处理流包装。

## 三、File类

在Java语言的java.io包中，由File类提供了描述文件和目录的操作与管理方法。但File类不是InputStream、OutputStream或Reader、Writer的子类，因为它不负责数据的输入输出，而专门用来管理磁盘文件与目录。

### 3.1、File 类的字段

![IO4](D:\notes\java-notes\资源\IO4.png)

### 3.2、File 构造方法

![IO5](D:\notes\java-notes\资源\IO5.png)

### 3.3、File 类的常用方法

1. 创建方法

boolean createNewFile() 不存在返回true 存在返回false
boolean mkdir() 创建目录，如果上一级目录不存在，则会创建失败
boolean mkdirs() 创建多级目录，如果上一级目录不存在也会自动创建

2. 删除方法

boolean delete() 删除文件或目录，如果表示目录，则目录下必须为空才能删除
boolean deleteOnExit() 文件使用完成后删除

3. 判断方法

boolean canExecute()判断文件是否可执行
boolean canRead()判断文件是否可读
boolean canWrite() 判断文件是否可写
boolean exists() 判断文件或目录是否存在
boolean isDirectory()  判断此路径是否为一个目录
boolean isFile()　　判断是否为一个文件
boolean isHidden()　　判断是否为隐藏文件
boolean isAbsolute()判断是否是绝对路径 文件不存在也能判断

4. 获取方法

String getName() 获取此路径表示的文件或目录名称
String getPath() 将此路径名转换为路径名字符串
String getAbsolutePath() 返回此抽象路径名的绝对形式
String getParent()//如果没有父目录返回null
long lastModified()//获取最后一次修改的时间
long length() 返回由此抽象路径名表示的文件的长度。
boolean renameTo(File f) 重命名由此抽象路径名表示的文件。
File[] liseRoots()//获取机器盘符
String[] list()  返回一个字符串数组，命名由此抽象路径名表示的目录中的文件和目录。
String[] list(FilenameFilter filter) 返回一个字符串数组，命名由此抽象路径名表示的目录中满足指定过滤器的文件和目录。

## 四、字节流InputStream/OutputStream

字节输入输出流：InputStream、OutputSteam

## 五、字符流Reader/Writer

字节输入输出流：Reader、Writer

## 六、包装流

①、包装流隐藏了底层节点流的差异，并对外提供了更方便的输入\输出功能，让我们只关心这个高级流的操作

②、使用包装流包装了节点流，程序直接操作包装流，而底层还是节点流和IO设备操作

③、关闭包装流的时候，只需要关闭包装流即可

### 6.1、缓冲流：是一个包装流，目的是缓存作用，加快读取和写入数据的速度

1. 字节缓冲流：BufferedInputStream、BufferedOutputStream
2. 字符缓冲流：BufferedReader、BufferedWriter

在操作流的时候通常都会定义一个字节或字符数组，将读取/写入的数据先存放到这个数组里面，然后在取数组里面的数据。这比我们一个一个的读取/写入数据要快很多，而这也就是缓冲流的由来。只不过缓冲流里面定义了一个 数组用来存储我们读取/写入的数据，当内部定义的数组满了（注意：我们操作的时候外部还是会定义一个小的数组，小数组放入到内部数组中），就会进行下一步操作。

![IO8](D:\notes\java-notes\资源\IO8.png)

### 6.2、转换流：把字节流转换为字符流

1. InputStreamReader:把字节输入流转换为字符输入流
2. OutputStreamWriter:把字节输出流转换为字符输出流

OutputStreamWriter和InputStreamReader是字符和字节的桥梁：也可以称之为字符转换流。字符转换流原理：字节流+编码表。

### 6.3、内存流（数组流）

1. 字节内存流：ByteArrayOutputStream 、ByteArrayInputStream
2. 字符内存流：CharArrayReader、CharArrayWriter
3. 字符串流：StringReader,StringWriter（把数据临时存储到字符串中）

把数据先临时存在数组中，也就是内存中。所以关闭 内存流是无效的，关闭后还是可以调用这个类的方法。

## 七、合并流

多个输入流合并为一个流，也叫顺序流，因为在读取的时候是先读第一个，读完了在读下面一个流。

原理解析：https://cloud.tencent.com/developer/article/1130044

```java
// 初始化需要合并的流
FileInputStream first = new FileInputStream(files[0]);
FileInputStream second = new FileInputStream(files[1]);
SequenceInputStream inputStream = new SequenceInputStream(first, second);
// 继续合并流
for (int i = 2; i < files.length; i++) {
    InputStream third = new FileInputStream(files[i]);
    inputStream = new SequenceInputStream(inputStream, third);
}
String finalPath = this.getFileRootPath() + updatePath + File.separator + this.getFileInfo().getFileName();
FileOutputStream outputStream = new FileOutputStream(finalPath);
try {
    byte[] bytes = new byte[1024];
    int len;
    // 读取数据
    while ((len = inputStream.read(bytes)) != -1) {
        outputStream.write(bytes, 0, len);
        outputStream.flush();
    }
} finally {
    inputStream.close();
    outputStream.close();
}
```

## 八、随机访问文件流

### 8.1、构造方法

![IO9](D:\notes\java-notes\资源\IO9.png)

"r" :    以只读方式打开。调用结果对象的任何 write 方法都将导致抛出 IOException。
 "rw":    打开以便读取和写入。
 "rws":  打开以便读取和写入。相对于 "rw"，"rws" 还要求对“文件的内容”或“元数据”的每个更新都同步写入到基础存储设备。
 **"**rwd**" :**  打开以便读取和写入，相对于 "rw"，"rwd" 还要求对“文件的内容”的每个更新都同步写入到基础存储设备。

### 8.2、适用场景

1. 适用于指定位置读取文件数据

需要指定读取偏移量，seek(long pos)方法跳过偏移量，从指定位置后边开始读数据

2. 适用于文件末尾追加写入/指定位置写入数据

文件末尾追加，先获取文件长度，seek方法跳过文件长

> 1. 指定位置写入：seek方法使光标指针跳跃到指定位置，光标指针后到文件结尾的数据缓存到临时文件(否则写入会被覆盖)
> 2. 然后写入新数据，再次追加结尾部分的临时数据

3. 大文件多线程读取下载

> 注意三种判断条件:
> 过度读取 : 读取总长度>需要下载的文件长度,停止读取
> 少量读取 : 读取总长度<需要下载的文件长度,继续正常读取
> 下载完成 : 停止读取









