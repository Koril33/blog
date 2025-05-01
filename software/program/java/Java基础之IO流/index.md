---
title: "Java基础之I/O流"
date: 2023-02-18T14:55:18+08:00
tags: []
featured_image: "images/background.jpg"
summary: "I/O API 是 Java 程序与外界沟通的桥梁"
toc: true
---

## 什么是I/O？

I/O（Input/Output）指的是输入和输出。Java I/O 的 API 构建了一个面向输入和输出的接口，输入和输出的内容是字节（也可以是字符或者其他格式的信息，不过从计算机本质上来看都是 0 和 1）。Java I/O 是所有应用的基石之一，它让 Java 程序有能力和外部进行信息交换，譬如：读取或写入磁盘文件内容，和外部数据库通讯（本质上是网络通讯）。

---

## I/O、NIO、NIO2

Java I/O API 在第一个 JDK 版本就发布了。

在 2002年，NIO 伴随着 JDK1.4 发布，提供了 Non-Blocking Input/Output。

在 2011年，NIO2 伴随着 JDK7 发布，在原有的 I/O API 上增加了很多新的功能。

---

## 流？

我们通常在 Java I/O 中讲到 IO 流的概念，这里的“流”类似于现实世界中的水流，电流等。

水流是水在水管中流动，电流是电子在导体中移动。那么 IO 流其实就是数据（0 和 1）在数字管道中流动了。

流动的过程一定有一个起点和终点，IO 流的起点和终点可以是磁盘，内存，网络，或者其他的程序。IO 流支持多种数据格式，包括最本质的字节（byte），基本数据类型（primitive data type），字符（character）或者对象（object）。一部分流只是简单的传送数据，一部分流可能会添加更高级的功能去操作，修改数据。

不管流的数据格式还是操作方式多么繁多，一个最基本的流程就是：Java 程序使用输入流（Input Stream）从源头（source）读入（read）数据，经过一些处理和操作（也可能不进行处理），最后使用输出流（Output Stream）将数据写出（write）到目的地（Destination）。

注意：这里的流和 Java8 中引入的 Stream API 不是一个东西，尽管名称一样。

---

## 字节 or 字符

Java IO 根据数据内容分成两部分 API：

1. 内容为字节，例如图片，视频，可执行文件这种二进制文件。
2. 内容为字符，例如文本文件，XML、JSON 文档这种人类可读文件。

面向字节的 API 顶层类是 InputStream 和 OutputStream。

面向字符的 API 顶层类是 Reader 和 Writer。

---

## 字节流

《深入理解计算机系统》开头提到了一句话：信息就是位 + 上下文。

计算机的所有内容，文件底层都是 0 和 1，8 个位构成了一个字节，理所当然的，我们先从面向字节的流开始说起。

Java IO 使用字节流（Byte Stream）来构建 8-bit 字节的输入（input）和输出（output）。所有的字节流都继承于 InputStream 和 OutputStream。

### FileInputStream 和 FileOutputStream

InputStream 和 OutputStream 有很多子类，它们虽然功能不一样，但是用法类似，这里将以最常见的 FileInputStream 和 FileOutputStream 举例，说明字节流的使用方法。

首先，我们使用 FileInputStream 从文件系统读取一个普通的文本文件内容到 Java 程序中，并在控制台上打印出来。

测试文件，poem.txt

```
Be through my lips to unawaken'd earth
The trumpet of a prophecy! Oh Wind,
If Winter comes, can Spring be far behind?
```

读取文件的代码，FileIOStreamTest.java

```java
import java.io.FileInputStream;
import java.io.IOException;

public class FileIOStreamTest {

    private static void showBytes(int b) {
        System.out.print("Num: " + b);
        System.out.println(", ASCII: " + (char) b);
    }

    public static void main(String[] args) {

        FileInputStream in = null;

        try {
            // 通过文件的路径字符串构造 FileInputStream 对象
            in = new FileInputStream("C:\\Users\\dingj\\Desktop\\poem.txt");
            int b;
            // 一次读取一个字节
            while ((b = in.read()) != -1) {
                // 按照特定格式打印在控制台上
                showBytes(b);
            }
        } catch (IOException e) {
            e.printStackTrace();
        } finally {
            // 如果 in 不为空，关闭文件流
            // 判断变量非空的原因是，如果此变量对应的文件流没有被正确打开，
            // 该变量的值不会发生改变（还是初始值 null），所以为了避免出现
            // NullPointerException 的异常，这里需要进行判空处理。
            if (in != null) {
                try {
                    in.close();
                } catch (IOException e) {
                    e.printStackTrace();
                }
            }
        }
    }
}
```

上面的代码看起来很长，实际上只干了一件事情，就是打开一个文本文件的字节流，从这个字节流里一个接一个读取单个字节，再打印到控制台上。

它的核心代码只有短短两三行，其他的代码都是在捕获异常（catch 语句块）和处理文件流的关闭（finally 语句块）。一旦打开的流有好几个，整个 try-catch-finally 语句就会变的非常的臃肿，一个套一个。

为了方便程序员开发，专注于业务逻辑，而不是绞尽脑汁思考如何正确的关闭每一个流，Java 7 提供了 try-with-resources 语句来取代这长的令人发指的嵌套 try-catch-finally 语句。编译器碰到整个语法糖就会自动生成相应的关闭流的代码，所以底层本质上还是  try-catch-finally，只是这个活不需要程序员来做了，交给了编译器自动处理。

使用了 try-with-resources 语句的代码如下：

```java
import java.io.FileInputStream;
import java.io.IOException;

public class FileIOStreamTest {

    private static void showBytes(int b) {
        System.out.print("Num: " + b);
        System.out.println(", ASCII: " + (char) b);
    }

    public static void main(String[] args) {

        try(FileInputStream in = new FileInputStream("C:\\Users\\dingj\\Desktop\\poem.txt")) {
            int b;
            while ((b = in.read()) != -1) {
                showBytes(b);
            }
        } catch (IOException e) {
            e.printStackTrace();
        }
    }
}
```

可以看到相比之前的代码简洁了不少，所谓的资源（resources）是指在程序完成后，必须关闭的对象。try-with-resources 语句确保了每个资源在语句结束时关闭。只有实现了 java.lang.AutoCloseable 接口才能用，FileInputStream 类继承自 InputStream 抽象类，而InputStream 抽象类实现了 Closeable 接口，而 Closeable 接口则继承自 AutoCloseable 接口，所以 FileInputStream 可以使用此语法糖。

将文件的字节流读入到 Java 程序中后，我们可以使用 FileOutputStream 将这些字节们写出到目的地中，这里的目的地指的是另外一个文件。

将 poem.txt 的所有字节，写入到 poem_dup.txt 中：

```java
package cn.korilweb;

import java.io.FileInputStream;
import java.io.FileOutputStream;
import java.io.IOException;

public class FileIOStreamTest {

    private static void showBytes(int b) {
        System.out.print("Num: " + b);
        System.out.println(", ASCII: " + (char) b);
    }

    public static void main(String[] args) {

        try (
            FileInputStream in = new FileInputStream("C:\\Users\\dingj\\Desktop\\poem.txt");
            FileOutputStream out = new FileOutputStream("C:\\Users\\dingj\\Desktop\\poem_dup.txt")
        ) {
            int b;
            while ((b = in.read()) != -1) {
                showBytes(b);
                // 将一个字节写到文件的字节输出流中
                out.write(b);
            }
        } catch (IOException e) {
            e.printStackTrace();
        }
    }
}
```

### 文档以及注意事项

InputStream 的 read 方法注释如下：

```
Reads the next byte of data from the input stream. The value byte is
returned as an {@code int} in the range {@code 0} to
{@code 255}. If no byte is available because the end of the stream
has been reached, the value {@code -1} is returned. This method
blocks until input data is available, the end of the stream is detected,
or an exception is thrown.
<p> A subclass must provide an implementation of this method.
@return     the next byte of data, or {@code -1} if the end of the
            stream is reached.
@throws     IOException  if an I/O error occurs.

翻译：
从输入流中读取数据的下一个字节。返回的字节值是 int 类型，其范围在 0-255 之间。
如果因为到达了流的末尾而没有下一个字节可读了，那将返回 -1。
此方法将一直阻塞，直到输入数据可用，或到达了流的末尾，或抛出了异常。
子类必须提供该方法的实现。
return 数据中下一个字节的值，如果到达了流的末尾，返回 -1。
throws 如果发生了 I/O 异常，则抛出 IOException。
```

OutputStream 的 write 方法注释如下：

```
Writes the specified byte to this output stream. The general
contract for {@code write} is that one byte is written
to the output stream. The byte to be written is the eight
low-order bits of the argument {@code b}. The 24
high-order bits of {@code b} are ignored.
<p>
Subclasses of {@code OutputStream} must provide an
implementation for this method.
@param      b   the {@code byte}.
@throws     IOException  if an I/O error occurs. In particular,
            an {@code IOException} may be thrown if the
            output stream has been closed.

翻译：
向输出流中写入指定的字节，write 的通常约定是写入一个字节到输出流中。
被写入的字节是形参的低 8 位。形参的高 24 位将被忽略。
OutputStream 的子类必须提供该方法的实现。
param 字节
throws 如果发生 I/O 错误，抛出 IOException。特别是，如果输出流已关闭，则可能会引发 IOException。
```

可以看到几个注意的地方，首先是 read 返回值和 write 形参的类型都是 int，明明写入和写出的都是字节，为什么要用 int 的低八位来表示一个 8-bits 字节呢？为什么不直接用 byte 类型呢？

因为 byte 类型的范围是从 -128~127（Java 的整形都是有符号整数，最高位是符号位），而约定遇到文件末尾，函数返回的 -1 指的是 int 类型的 -1，即 0xFF FF FF FF，byte 类型的 -1 是 0xFF。

倘若 InputStream 的 read 方法返回的是 byte 类型的话，重新审视以下代码：

```java
byte b;
while ((b = in.read()) != -1) {
    out.write(b);
}
```

遇到了文件中 “1111 1111” 的字节，b 将会被解释成 -1 而不是我们希望的 255，不再符合“!= -1”的条件就会退出 while 循环（但根本没有到达文件的末尾）。

而采用 int 类型作为返回值：

```java
int b;
while ((b = in.read()) != -1) {
    out.write(b);
}
```

此时，遇到了文件中的 “1111 1111” 的字节，b 将会被解释成正常的 255，用 int 类型的二进制表示为 “0000 0000 0000 0000 0000 0000 1111 1111”，与 -1 比较（int 类型的 -1 用二进制表示为 “1111 1111 1111 1111 1111 1111 1111 1111”，明显是不相等的，循环将会正常进行。

也就是说，不用 byte 是因为无法正常的表示 255，会被解释成 byte 类型的 -1，然后误以为到达文件末尾，终止了循环，但 short 似乎也可以表示 255，之所以不用 short 而用 int，可以参考下面的帖子：

https://stackoverflow.com/questions/21062744/why-does-inputstream-read-return-an-int-and-not-a-short

---

## 字节流的缓存

### InputStream 和 OutputStream 中的缓存数组

上一小结的 read() 和 write() 底层调用的 native 方法，一次返回或者写入一个字节，效率低下，翻看 InputStream 的 API 可以看到还有另外一个有关字节数组的方法：

```java
public int read(byte b[], int off, int len) throws IOException {
    Objects.checkFromIndexSize(off, len, b.length);
    if (len == 0) {
        return 0;
    }

    int c = read();
    if (c == -1) {
        return -1;
    }
    b[off] = (byte)c;

    int i = 1;
    try {
        for (; i < len ; i++) {
            c = read();
            if (c == -1) {
                break;
            }
            b[off + i] = (byte)c;
        }
    } catch (IOException ee) {
    }
    return i;
}
```

该方法接收三个参数，缓存数组 b 存放读取的字节内容，off 表示从数组 b 的什么位置开始存储，len 表示读取的最长字节数量。返回一个 int 类型的参数，表示读取到缓存数组的字节数量。

同样的，OutputStream 也有对称的 write(byte[] b, int off, int len) 的方法，一次写入一个字节数组的内容。

使用这两个带缓存的方法可以一次读取和写入一个数组的字节数据，下面是改写后的代码：

```java
package cn.korilweb;

import java.io.FileInputStream;
import java.io.FileOutputStream;
import java.io.IOException;

public class FileIOStreamTest {

    private static void showBytes(int b) {
        System.out.print("Num: " + b);
        System.out.println(", ASCII: " + (char) b);
    }

    private static void showBytes(byte[] bytes) {
        for (byte b : bytes) {
            showBytes(b);
        }
    }

    public static void main(String[] args) {
		byte[] buffer = new byte[1024];
        int readLen;
        try (
            FileInputStream in = new FileInputStream("C:\\Users\\dingj\\Desktop\\poem.txt");
            FileOutputStream out = new FileOutputStream("C:\\Users\\dingj\\Desktop\\poem_dup.txt")
        ) {
            while ((readLen = in.read(buffer)) != -1) {
                showBytes(buffer);
                out.write(buffer, 0, readLen);
            }
        } catch (IOException e) {
            e.printStackTrace();
        }
    }
}
```

### BufferedInputStream 和 BufferedOutputStream

除了以上通过自己构造字节数组来作为缓存，Java 也提供了缓存类，从名字上来看，就知道这两个类主要负责提供缓存机制，以提高读写效率。这里用到的设计模式——装饰者模式（Decorator Pattern）。

装饰者模式，通过创建一个新的包装类（BufferedInputStream），在不修改原来类（FileInputStream）的情况下，扩展一个类的功能（增加缓存功能）。

BufferedInputStream 和 BufferedOutputStream 继承于 FilterInputStream 和 FilterOutputStream，后两者继承于 InputStream 和 OutputStream。

使用这两个类的 read 和 write 方法时，在循环中就不是反复调用底层的 native 方法，而是从这两个类的 buf 数组中获取或写入字节信息。相当于在使用者和 native 方法中间加了个缓存数组。

修改后的代码：

```java
public static void main(String[] args) {
    try (
        BufferedInputStream in = new BufferedInputStream(new FileInputStream("C:\\Users\\dingj\\Desktop\\poem.txt"));
        BufferedOutputStream out = new BufferedOutputStream(new FileOutputStream("C:\\Users\\dingj\\Desktop\\poem_dup.txt"))
    ) {
        int b;
        while ((b = in.read()) != -1) {
            showBytes(b);
            out.write(b);
        }
    } catch (IOException e) {
        e.printStackTrace();
    }
}
```

### 缓存和不缓存的性能区别

为了直观的体现原始 InputStream 读写和使用了 BufferedInputStream 读写速度的差异，我写了个简单的测试代码，以一张图片（7.61M，7,982,499 字节）作为拷贝对象。

测试代码：

```java
package cn.korilweb;

import java.io.*;
import java.nio.file.Files;

public class FileIOStreamTest {
    /**
     * 不带 Buffer 的文件流
     */
    public static void duplicate1(String src, String des) {
        try (
            FileInputStream srcFile = new FileInputStream(src);
            FileOutputStream desFile = new FileOutputStream(des);
        ) {
            int c;
            while ((c = srcFile.read()) != -1) {
                desFile.write(c);
            }

        } catch (IOException e) {
            e.printStackTrace();
        }
    }


    /**
     * 使用了 BufferedInputStream 包装的文件流
     */
    public static void duplicate2(String src, String des) {
        try (
            BufferedInputStream bufferedSrcFile = new BufferedInputStream(
                    new FileInputStream(src),
                    1024 * 4
            );
            BufferedOutputStream bufferedDesFile = new BufferedOutputStream(
                    new FileOutputStream(des),
                    1024 * 4
            )
        ) {
            int c;
            while ((c = bufferedSrcFile.read()) != -1) {
                bufferedDesFile.write(c);
            }

        } catch (IOException e) {
            e.printStackTrace();
        }
    }


    /**
     * 使用 InputStream 中带缓存数组方法的文件流
     */
    public static void duplicate3(String src, String des, int len) {
        byte[] buffer = new byte[len];
        int readCnt;
        try (
            FileInputStream bufferedSrcFile = new FileInputStream(src);
            FileOutputStream bufferedDesFile = new FileOutputStream(des)
        ) {
            while ((readCnt = bufferedSrcFile.read(buffer)) != -1) {
                bufferedDesFile.write(buffer, 0, readCnt);
            }

        } catch (IOException e) {
            e.printStackTrace();
        }
    }

    /**
     * 使用了 BufferedInputStream 包装的文件流
     * 并且调用 read(byte[] b) 的方法
     */
    public static void duplicate4(String src, String des, int len) {
        byte[] buffer = new byte[len];
        int readCnt;
        try (
            BufferedInputStream in = new BufferedInputStream(new FileInputStream(src));
            BufferedOutputStream out = new BufferedOutputStream(new FileOutputStream(des))
        ) {
            while ((readCnt = in.read(buffer)) != -1) {
                out.write(buffer, 0, readCnt);
            }
        } catch (IOException e) {
            e.printStackTrace();
        }
    }


    public static void main(String[] args) {

        String src = "C:\\Users\\dingj\\Desktop\\img.jpg";
        String des = "C:\\Users\\dingj\\Desktop\\des";

        long start1 = System.currentTimeMillis();
        duplicate1(src, des + "\\img_1.jpg");
        long end1 = System.currentTimeMillis();

        System.out.println("no buffered cost: " + (end1 - start1) + " ms");

        long start2 = System.currentTimeMillis();
        duplicate2(src, des + "\\img_2.jpg");
        long end2 = System.currentTimeMillis();
        System.out.println("BufferedInputStream cost: " + (end2 - start2) + " ms");


        for (int i = 1; i <= 64; i++) {
            long start = System.currentTimeMillis();
            duplicate3(src, des + "\\img_a_" + i + ".jpg", i * 1024);
            long end = System.currentTimeMillis();
            System.out.println("buffered array cost" + "-" + i + ": " + (end - start) + " ms");
        }

        for (int i = 1; i <= 64; i++) {
            long start = System.currentTimeMillis();
            duplicate4(src, des + "\\img_b_" + i + ".jpg", i * 1024);
            long end = System.currentTimeMillis();
            System.out.println("buffered and array cost" + "-" + i + ": " + (end - start) + " ms");
        }
    }
}
```

测试结果：

```
no buffered cost: 64526 ms
BufferedInputStream cost: 544 ms
buffered array cost-1: 140 ms
buffered array cost-2: 76 ms
buffered array cost-3: 47 ms
buffered array cost-4: 28 ms
buffered array cost-5: 30 ms
buffered array cost-6: 37 ms
buffered array cost-7: 41 ms
buffered array cost-8: 26 ms
buffered array cost-9: 44 ms
buffered array cost-10: 27 ms
buffered array cost-11: 142 ms
buffered array cost-12: 54 ms
buffered array cost-13: 18 ms
buffered array cost-14: 62 ms
buffered array cost-15: 28 ms
buffered array cost-16: 79 ms
buffered array cost-17: 26 ms
buffered array cost-18: 17 ms
buffered array cost-19: 16 ms
buffered array cost-20: 10 ms
buffered array cost-21: 23 ms
buffered array cost-22: 32 ms
buffered array cost-23: 19 ms
buffered array cost-24: 68 ms
buffered array cost-25: 45 ms
buffered array cost-26: 14 ms
buffered array cost-27: 24 ms
buffered array cost-28: 15 ms
buffered array cost-29: 11 ms
buffered array cost-30: 11 ms
buffered array cost-31: 11 ms
buffered array cost-32: 10 ms
buffered array cost-33: 11 ms
buffered array cost-34: 11 ms
buffered array cost-35: 9 ms
buffered array cost-36: 8 ms
buffered array cost-37: 9 ms
buffered array cost-38: 8 ms
buffered array cost-39: 12 ms
buffered array cost-40: 8 ms
buffered array cost-41: 9 ms
buffered array cost-42: 10 ms
buffered array cost-43: 8 ms
buffered array cost-44: 10 ms
buffered array cost-45: 17 ms
buffered array cost-46: 12 ms
buffered array cost-47: 9 ms
buffered array cost-48: 16 ms
buffered array cost-49: 11 ms
buffered array cost-50: 13 ms
buffered array cost-51: 13 ms
buffered array cost-52: 9 ms
buffered array cost-53: 8 ms
buffered array cost-54: 12 ms
buffered array cost-55: 8 ms
buffered array cost-56: 13 ms
buffered array cost-57: 11 ms
buffered array cost-58: 13 ms
buffered array cost-59: 10 ms
buffered array cost-60: 16 ms
buffered array cost-61: 12 ms
buffered array cost-62: 11 ms
buffered array cost-63: 16 ms
buffered array cost-64: 20 ms
buffered and array cost-1: 33 ms
buffered and array cost-2: 40 ms
buffered and array cost-3: 64 ms
buffered and array cost-4: 21 ms
buffered and array cost-5: 39 ms
buffered and array cost-6: 36 ms
buffered and array cost-7: 25 ms
buffered and array cost-8: 31 ms
buffered and array cost-9: 18 ms
buffered and array cost-10: 19 ms
buffered and array cost-11: 16 ms
buffered and array cost-12: 14 ms
buffered and array cost-13: 14 ms
buffered and array cost-14: 13 ms
buffered and array cost-15: 22 ms
buffered and array cost-16: 22 ms
buffered and array cost-17: 18 ms
buffered and array cost-18: 14 ms
buffered and array cost-19: 14 ms
buffered and array cost-20: 15 ms
buffered and array cost-21: 43 ms
buffered and array cost-22: 118 ms
buffered and array cost-23: 21 ms
buffered and array cost-24: 27 ms
buffered and array cost-25: 20 ms
buffered and array cost-26: 19 ms
buffered and array cost-27: 10 ms
buffered and array cost-28: 10 ms
buffered and array cost-29: 10 ms
buffered and array cost-30: 9 ms
buffered and array cost-31: 9 ms
buffered and array cost-32: 11 ms
buffered and array cost-33: 14 ms
buffered and array cost-34: 10 ms
buffered and array cost-35: 11 ms
buffered and array cost-36: 11 ms
buffered and array cost-37: 11 ms
buffered and array cost-38: 10 ms
buffered and array cost-39: 11 ms
buffered and array cost-40: 11 ms
buffered and array cost-41: 14 ms
buffered and array cost-42: 12 ms
buffered and array cost-43: 9 ms
buffered and array cost-44: 13 ms
buffered and array cost-45: 25 ms
buffered and array cost-46: 7 ms
buffered and array cost-47: 11 ms
buffered and array cost-48: 8 ms
buffered and array cost-49: 16 ms
buffered and array cost-50: 7 ms
buffered and array cost-51: 8 ms
buffered and array cost-52: 18 ms
buffered and array cost-53: 7 ms
buffered and array cost-54: 8 ms
buffered and array cost-55: 7 ms
buffered and array cost-56: 13 ms
buffered and array cost-57: 16 ms
buffered and array cost-58: 11 ms
buffered and array cost-59: 19 ms
buffered and array cost-60: 13 ms
buffered and array cost-61: 15 ms
buffered and array cost-62: 22 ms
buffered and array cost-63: 9 ms
buffered and array cost-64: 6 ms
```

可以看到后三者使用了缓存方案，明显比第一个不用缓存的方法，节省了很多时间，而自己写缓存数组的方法似乎比使用 BufferedInputStream 更快，但不意味着实践中采用自己编写的缓存数组方案，因为很容易出错，具体的讨论可以参考这篇文章：https://www.oracle.com/technical-resources/articles/javase/perftuning.html

---

## 字符流

讨论完最基础的字节流之后，我们将目光转向另外一个分支——字符流。平时面向文本类文件操作时，我们应该选择字符流的 API，因为它们能提供更加多元的方法，相比之下，字节流工作在最底层的 I/O，其他的（包括字符流）流都是基于最底层的字节流而构建的。

Java 平台使用 Unicode 存储字符，准确的说是使用 UTF-16 编码。字符流会自动将原始字节与本地字符集相互转换。关于 Unicode 的讲解，可以参考我之前的文章：https://korilweb.cn/tech/%E8%81%8A%E8%81%8Aunicodeutf-8utf-16%E4%BB%A5%E5%8F%8Autf-32%E7%9A%84%E9%82%A3%E4%BA%9B%E4%BA%8B%E5%84%BF/

和字节流中的 InputStream 相对应的字符输入流是 Reader，和字节流中的 OutputStream 相对应的字符输出流是 Writer。同样的，对应的文件流分别是 FileReader 和 FileWriter。

### FileReader 和 FileWriter

我将 poem.txt 的英文内容换成中文内容：

```
愿中国青年都摆脱冷气，只是向上走，不必听自暴自弃者流的话。
能做事的做事，能发声的发声。有一分热，发一分光，就令萤火一般，也可以在黑暗里发一点光，不必等候炬火。
此后如竟没有炬火：我便是唯一的光。
```

同样是读取 poem.txt 并且将所有内容写入到 poem_dup.txt 中，代码几乎是不需要改动的：

```java
package cn.korilweb;

import java.io.*;

public class FileIOStreamTest {
    public static void main(String[] args) {

        try (
            FileReader reader = new FileReader("C:\\Users\\dingj\\Desktop\\poem.txt");
            FileWriter writer = new FileWriter("C:\\Users\\dingj\\Desktop\\poem_dup.txt")
        ) {
            int c;
            while ((c = reader.read()) != -1) {
                writer.write(c);
            }
        } catch (IOException e) {
            e.printStackTrace();
        }
    }
}
```

查看 FileReader 和 FileWriter 的源码可以发现，字符流通常就是字节流的“包装器”。字符流使用字节流执行底层的物理 I/O，而字符流则负责处理字符和字节之间的转换，FileReader 底层代码使用 FileInputStream，而 FileWriter 使用 FileOutputStream。

我截去了第一行的打印内容，如下：

```
num: 24895, char: 愿
num: 20013, char: 中
num: 22269, char: 国
num: 38738, char: 青
num: 24180, char: 年
num: 37117, char: 都
num: 25670, char: 摆
num: 33073, char: 脱
num: 20919, char: 冷
num: 27668, char: 气
num: 65292, char: ，
num: 21482, char: 只
num: 26159, char: 是
num: 21521, char: 向
num: 19978, char: 上
num: 36208, char: 走
num: 65292, char: ，
num: 19981, char: 不
num: 24517, char: 必
num: 21548, char: 听
num: 33258, char: 自
num: 26292, char: 暴
num: 33258, char: 自
num: 24323, char: 弃
num: 32773, char: 者
num: 27969, char: 流
num: 30340, char: 的
num: 35805, char: 话
num: 12290, char: 。
num: 13, char: 
num: 10, char: 

```

可以看到与英文字符不同的是，中文字符的 Unicode 码很大，超过了之前字节流中提到的范围。查看 Reader 的源码文档发现 reader 的范围是 0-65535（16进制的 0x00-0xFFFF），InputStream 中使用 int 类型的低 8 位存储字节值，那么这里的 Reader 就是用 int 类型的低 16 位存储字符值。

---

## 字符流的缓存

字符流也有负责缓存的包装类：BufferedReader 和 BufferedWriter。

用法和之前的类似，不再赘述

```java
package cn.korilweb;

import java.io.*;

public class FileIOStreamTest {

    private static void showChar(int c) {
        System.out.print("num: " + c);
        System.out.println(", char: " + (char) c);
    }


    public static void main(String[] args) {

        try (
            BufferedReader reader = new BufferedReader(new FileReader("C:\\Users\\dingj\\Desktop\\poem.txt"));
            BufferedWriter writer = new BufferedWriter(new FileWriter("C:\\Users\\dingj\\Desktop\\poem_dup.txt"))
        ) {
            int c;
            while ((c = reader.read()) != -1) {
                showChar(c);
                writer.write(c);
            }
        } catch (IOException e) {
            e.printStackTrace();
        }
    }
}
```

---

## 面向单词和行

上面提到的字节和字符依然相对原始，平时的编程中，大部分会更多涉及有结构的可读性的数据，比方说处理一个文本文件，统计它的单词的个数，行数，那么仅仅使用字节和字符就有些难办了。

### Scanning

Scanner 类型的对象可用于将格式化输入分解为标记（Token），并根据其数据类型转换单个标记。

默认情况下，扫描仪使用空格来分隔标记。（空白字符包括空格、制表符和行终止符。有关完整列表，请参阅 [`Character.isWhitespace`](https://docs.oracle.com/javase/8/docs/api/java/lang/Character.html#isWhitespace-char-) 的文档）。下面我们将 poem.txt 中的英文单词一个一个读出来。

```java
public static void main(String[] args) {

    try (
        Scanner scanner = new Scanner(new BufferedReader(new FileReader("C:\\Users\\dingj\\Desktop\\poem.txt")))
    ) {
        String w;
        while (scanner.hasNext()) {
            w = scanner.next();
            System.out.println(w);
        }
    } catch (FileNotFoundException e) {
        e.printStackTrace();
    }
}
```

打印结果：

```
Be
through
my
lips
to
unawaken'd
earth
The
trumpet
of
a
prophecy!
Oh
Wind,
If
Winter
comes,
can
Spring
be
far
behind?
```

如果想使用其他分隔符，可以调用 useDelimiter 方法，传入正则表达式：

```java
public static void main(String[] args) {

    try (
        Scanner scanner = new Scanner("this,is,a,test")
    ) {
        scanner.useDelimiter(",");
        String w;
        while (scanner.hasNext()) {
            w = scanner.next();
            System.out.println(w);
        }
    }
}
```

### BufferedReader 的 readLine

单词可以依靠 Scanner 提供的方法分割，那么句子也是可以的（scanner.useDelimiter("\r\n");），除了 Scanner 之外，句子的分割读取可以使用 BufferedReader 提供的 readLine 方法：

```java
public static void main(String[] args) {

    try (
        BufferedReader reader = new BufferedReader(new FileReader("C:\\Users\\dingj\\Desktop\\poem.txt"))
    ) {
        String l;
        int cnt = 1;
        while ((l = reader.readLine()) != null) {
            System.out.println(cnt + ". " + l);
            cnt++;
        }
    } catch (IOException e) {
        e.printStackTrace();
    }
}
```

输出结果：

```
1. Be through my lips to unawaken'd earth
2. The trumpet of a prophecy! Oh Wind,
3. If Winter comes, can Spring be far behind?
```

需要注意的是，这里 readLine 判断文件到达末尾，返回的不再是 -1 而是 null。

## Data Streams



## Object Streams



---

## 参考

1.  https://docs.oracle.com/javase/tutorial/essential/io/index.html
2. https://dev.java/learn/java-io/intro/
3. https://jenkov.com/tutorials/java-io/index.html
