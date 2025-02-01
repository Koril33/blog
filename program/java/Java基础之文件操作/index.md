---
title: "Java基础之文件操作"
date: 2023-02-18T14:55:48+08:00
tags: []
featured_image: "images/background.jpg"
summary: "围绕 Files 和 Path 这两个类的介绍"
toc: true
---

## Path 类

Path 是在 Java NIO2 更新时加入的（Java SE7），完全限定名称是：java.nio.file.Path。Path 用来表示文档系统中的路径。路径可以指向文档或目录。路径可以是绝对路径，也可以是相对路径。

绝对路径包含从文档系统根目录到它指向的文档或目录的完整路径。相对路径包含相对于某个其他路径的文档或目录的路径。

## 创建 Path 实例对象

### Paths.get or Path.of ?

Path 有个静态工厂方法类，限定名叫：java.nio.file.Paths。Paths 比 Path 多了个 s，这种多带了个 s 的方法一般都是静态工厂（Files，FileSystems 也差不多），Paths 里面有两个静态方法，用来构造 Path：

| Modifier and Type | Method                              | Description                                                  |
| :---------------- | ----------------------------------- | ------------------------------------------------------------ |
| `static Path`     | `get(String first, String... more)` | Converts a path string, or a sequence of strings that when joined form a path string, to a `Path`. |
| `static Path`     | `get(URI uri)`                      | Converts the given URI to a [`Path`](Path.html) object.      |

本文只关注第一个方法，其实 get 的源码相当简单：

```java
public static Path get(String first, String... more) {
    // 调用 Path 的 of 方法
    return Path.of(first, more);
}
```

很多教程（包括《Java核心技术卷》）都是这么写的：

```java
Path p = Paths.get("C:\\text.txt");
```

我想这些教程这么写大概有两个原因：一个是懒，不想解释两个类的关系，第二个是考虑到兼容性，毕竟不是每个人都能用得上新版本的Java😂。

那么，Path.of 和 Paths.get 有什么区别呢？Path 是在 Java7 中被引入的，而在那个版本，接口中的静态方法还不存在，所以每个接口（比如这里的 Path）都需要一个伴生类（或者说静态工厂类）来提供一些静态方法，但是在 Java 8 之后，接口中可以放静态方法了，这个方法就没有什么存在的必要了，我们看看 Paths 的注释，就能略窥一二：

> API 说明:
> It is recommended to obtain a Path via the Path.of methods instead of via the get methods defined in this class as this class may be deprecated in a future release.
>
> 翻译：
>
> 建议通过 Path.of 方法而不是通过此类中定义的 get 方法获取路径，因为此类可能在未来版本中被弃用。

所以，不管是从官方给的注释来看，还是为了平时写代码的简便（创建一个Path，还需要额外引入一个类）而言，选择下面这种方式更加简单、快捷：

```java
Path p = Path.of("C:\\text.txt");
```

关于 Path.of 和 Paths.get 的讨论可以看下面两个链接：

* https://www.baeldung.com/java-paths-get-path-of
* https://stackoverflow.com/questions/58631724/paths-get-vs-path-of

### Path.of

Path.of 的源码如下：

```java
public static Path of(String first, String... more) {
    return FileSystems.getDefault().getPath(first, more);
}
```

后面的 more 是个可变参数，如果不为空的话，就会拼接成一串完整的路径：

```java
Path p = Path.of("C:\\Users\\dingj\\Desktop", "test_dir", "test_file");
System.out.println(p.toAbsolutePath());
```

打印结果：

```
C:\Users\dingj\Desktop\test_dir\test_file
```

### 相对路径

上面的例子都是绝对路径，从根目录出发，Path.of 也支持相对路径，相对路径的完整路径（绝对路径）是通过将基本路径与相对路径组合得出的。相对路径有两个特殊的标记符号：

* .
* ..

一个点代表当前路径，两个点表示上一级目录，比如：

```java
Path currentPath = Path.of(".", "dir1");
Path parentPath = Path.of("..", "dir2");
System.out.println(currentPath.toAbsolutePath());
System.out.println(parentPath.toAbsolutePath());
```

打印结果：

```
E:\StudyCode\Java\file\file\.\dir1
E:\StudyCode\Java\file\file\..\dir2
```

当然这样看不出啥名堂，试试创建文件夹，就明显看出二者差别了：

```java
Path currentPath = Path.of(".", "dir1");
Path parentPath = Path.of("..", "dir2");
Files.createDirectories(currentPath);
Files.createDirectories(parentPath);
```

结果发现，dir1 创建在了当前目录下，dir2 创建在了上级目录下。

### Path.resolve

resolve 的主要功能就是拼接两个 Path，针对“已有的 Path”去解析另外一个“给定的 Path”，最后返回一个结果 Path。

“给定的 Path”作为参数，需要区分三种情况：

1. 给定的 Path 为空，则返回当前 Path。
2. 给定的 Path 为绝对路径，则返回这个给定的 Path。
3. 除上两种情况，则返回已有 Path 和给定的 Path 的解析结果。

示例：

```java
// 已有的 path
Path basePath = Path.of("C:\\Users\\dingj\\Desktop");
// 第一种情况，给定的 path 为空路径
Path otherEmptyPath = Path.of("");
// 第二种情况，给定的 path 为绝对路径
Path otherAbsolutePath = Path.of("D:\\");
// 第三种情况
Path otherPath = Path.of("dir");

Path p1 = basePath.resolve(otherEmptyPath);
Path p2 = basePath.resolve(otherAbsolutePath);
Path p3 = basePath.resolve(otherPath);

System.out.println(p1.toAbsolutePath());
System.out.println(p2.toAbsolutePath());
System.out.println(p3.toAbsolutePath());
```

打印结果：

```
C:\Users\dingj\Desktop
D:\
C:\Users\dingj\Desktop\dir
```

### Path.resolveSibling

resolveSibling 在同一目录下的重命名相当有用，所以相比于 resolve 的拼接，resolveSibling 更像是一种替换：

```java
// 已有的 path
Path file = Path.of("C:\\Users\\dingj\\Desktop\\file.txt");
Path siblingFile = file.resolveSibling("sibling_file.txt");
System.out.println("file: " + file.toAbsolutePath());
System.out.println("sibling file: " + siblingFile.toAbsolutePath());
```

打印结果：

```
file: C:\Users\dingj\Desktop\file.txt
sibling file: C:\Users\dingj\Desktop\sibling_file.txt
```

可以看到，file 和 siblingFile 都具有相同的父级目录。

## 获取路径中的元素

* getRoot 用于获取根路径
* getParent 用于获取父级路径
* getNameCount 用于获取路径中元素的总个数
* getName 根据给定的 index，获取路径中元素的名称

示例：

```java
Path file = Path.of("C:\\Users\\dingj\\Desktop\\file.txt");
// 根路径
System.out.println(file.getRoot().toAbsolutePath());
// 文件名，离根目录最远的那个元素
System.out.println(file.getFileName());
// 父级路径
System.out.println(file.getParent());
// 路径中元素的个数
System.out.println(file.getNameCount());

for (int i = 0; i < file.getNameCount(); i++) {
    System.out.println(i + ": " + file.getName(i));
}
```

---

## Files 类

java.nio.file.Files 包含了很多用来操作文件和目录的静态方法，本文简单的介绍其中常用的一些操作，在了解 Files 之前，必须对 java.nio.file.Path 有所了解，因为 Files 大部分静态方法都是对给定的 Path 做一些操作。

## java.nio.file.Files or java.io.File?

这两个类从名字来说听起来挺像，就像是 Path 和 Paths，但实际上 java.io.File 是自 Java1.0 以来最老的一个对于文件和目录进行抽象的类，所以从功能上来说，它和 java.nio.file.Path 更相似。

但是我们从下面这段 java.io.File 文件里开头的注释可以看到，最好用 java.nio.file.Files 和 java.nio.file.Path 这两者的组合来替代 java.io.File。

>The java.nio.file package defines interfaces and classes for the Java virtual machine to access files, file attributes, and file systems. This API may be used to overcome many of the limitations of the java.io.File class. The toPath method may be used to obtain a Path that uses the abstract path represented by a File object to locate a file. The resulting Path may be used with the java.nio.file.Files class to provide more efficient and extensive access to additional file operations, file attributes, and I/O exceptions to help diagnose errors when an operation on a file fails.
>
>翻译：
>
>java.nio.file 包定义了 Java 虚拟机访问文档、文档属性和文档系统的接口和类。此 API 可用于克服 java.io.File 类的许多限制。toPath 方法可用于获取一个 Path，该路径使用 File 对象表示的抽象路径来查找文档。生成的 Path 可以与 java.nio.file.Files 类一起使用，以提供对其他文档操作、文档属性和 I/O 异常的更有效和更广泛的访问，以帮助在文档操作失败时诊断错误。

---

## 创建新的文件或目录

### Files.createFile

createFile 用于创建一个空文件，如果文件已经存在了，就会抛出一个异常。

```java
Path file = Path.of("C:\\Users\\dingj\\Desktop\\file.txt");
Files.createFile(file);
```

执行完以后，`C:\Users\dingj\Desktop\`路径下就会出现一个名为`file.txt`的文件

如果再执行一次这个代码，就会抛出 FileAlreadyExistsException 异常，因为 file.txt 已经存在了：

```
Exception in thread "main" java.nio.file.c: C:\Users\dingj\Desktop\file.txt
	at java.base/sun.nio.fs.WindowsException.translateToIOException(WindowsException.java:87)
	at java.base/sun.nio.fs.WindowsException.rethrowAsIOException(WindowsException.java:103)
	at java.base/sun.nio.fs.WindowsException.rethrowAsIOException(WindowsException.java:108)
	at java.base/sun.nio.fs.WindowsFileSystemProvider.newByteChannel(WindowsFileSystemProvider.java:231)
	at java.base/java.nio.file.Files.newByteChannel(Files.java:370)
	at java.base/java.nio.file.Files.createFile(Files.java:647)
	at cn.korilweb.Main.main(Main.java:20)
```

### Files.createDirectory

createDirectory 可以创建一个目录，同样的，如果文件已经存在了，就会抛出一个异常。

```java
Path file = Path.of("C:\\Users\\dingj\\Desktop\\dir");
Files.createDirectory(file);
```

### Files.createDirectories

createDirectory 只能在已有的目录下，创建一个新的目录，无法支持创建多级目录，假如父级目录不存在，就会抛异常：

```java
Path file = Path.of("C:\\Users\\dingj\\Desktop\\dir1\\dir2");
Files.createDirectory(file);
```

这里如果父目录`C:\\Users\\dingj\\Desktop\\dir1`目录不存在，会抛出 NoSuchFileException 异常：

```
Exception in thread "main" java.nio.file.NoSuchFileException: C:\Users\dingj\Desktop\dir1\dir2
	at java.base/sun.nio.fs.WindowsException.translateToIOException(WindowsException.java:85)
	at java.base/sun.nio.fs.WindowsException.rethrowAsIOException(WindowsException.java:103)
	at java.base/sun.nio.fs.WindowsException.rethrowAsIOException(WindowsException.java:108)
	at java.base/sun.nio.fs.WindowsFileSystemProvider.createDirectory(WindowsFileSystemProvider.java:505)
	at java.base/java.nio.file.Files.createDirectory(Files.java:689)
	at cn.korilweb.Main.main(Main.java:20)
```

使用 createDirectories 可以解决这个问题，它能创建多级目录：

```java
Path file = Path.of("C:\\Users\\dingj\\Desktop\\dir1\\dir2");
Files.createDirectories(file);
```

## 文件存在，拷贝，移动和删除

### Files.exists 和 Files.notExists

exists 用于判断给定的 Path 是否存在，notExists 判断给定的 Path 是否不存在：

```java
Path p = Path.of("C:\\Users\\dingj\\Desktop");
System.out.println(Files.exists(p)); // true
System.out.println(Files.notExists(p)); // false
```

所以，创建一个文件（不知道文件存不存在，但文件存在的话，就不创建新的），可以这么写：

```java
Path p = Path.of("C:\\Users\\dingj\\Desktop\\test.txt");
if (Files.notExists(p)) {
    System.out.println("file not exist, so create a new one.");
    Files.createFile(p);
} else {
    System.out.println("file exists, do not need to create file.");
}
```

先判断存在与否，再创建目录也是类似的写法：

```java
Path p = Path.of("C:\\Users\\dingj\\Desktop\\test\\foo\\bar");
if (Files.notExists(p)) {
    System.out.println("dir not exist, so create a new one.");
    Files.createDirectories(p);
} else {
    System.out.println("dir exists, do not need to create dir.");
}
```

### Files.copy

把一个文件从某一个路径下，拷贝到另外一个路径可以用 copy：

```java
Path src = Path.of("C:\\Users\\dingj\\Desktop\\file.txt");
Path des = Path.of("./file.txt");
// 把桌面下的某个文件拷贝到当前目录下
Files.copy(src, des);
```

如果再重复执行一遍上面的代码，则会抛出 FileAlreadyExistsException 的异常，如果我们希望拷贝的时候，如果已经存在该文件了，进行覆盖，则需要添加第三个参数：

```java
Path src = Path.of("C:\\Users\\dingj\\Desktop\\file.txt");
Path des = Path.of("./file.txt");
Files.copy(src, des, StandardCopyOption.REPLACE_EXISTING);
```

添加了 StandardCopyOption.REPLACE_EXISTING 以后，会直接覆盖，而不会再抛出异常了。

copy 可以复制目录。但是，目录中的文件不会被复制，因此即使原始目录包含文件，新目录也是空的。

除了文件复制之外，Files 还定义了可用于在文件和流之间进行复制的方法。copy(InputStream, Path, CopyOptions...)方法可用于将所有字节从输入流复制到文件。copy(Path, OutputStream)方法可用于将所有字节从文件复制到输出流。

### Files.move

move 可以用来移动或者重命名文件：

```java
Path src = Path.of("C:\\Users\\dingj\\Desktop\\file.txt");
Path des = Path.of("./file.txt");
Files.move(src, des);
```

原来目录下的文件就消失了，移动到了当前目录下。

同样的，如果已经存在了，会抛出 FileAlreadyExistsException 的异常，添加 StandardCopyOption.REPLACE_EXISTING 以后，会直接进行覆盖：

```java
Path src = Path.of("C:\\Users\\dingj\\Desktop\\file.txt");
Path des = Path.of("./file.txt");
Files.move(src, des, StandardCopyOption.REPLACE_EXISTING);
```

### Files.delete 和 Files.deleteIfExists

delete 用来删除一个文件或者空目录：

```java
Path src = Path.of("C:\\Users\\dingj\\Desktop\\file.txt");
Files.delete(src);
```

如果文件不存在，会抛出 NoSuchFileException 的异常，Files 提供了另外一个方法——deleteIfExists：

```java
Path src = Path.of("C:\\Users\\dingj\\Desktop\\file.txt");
boolean b = Files.deleteIfExists(src);
```

和返回 void 的 delete 不同，deleteIfExists 返回一个布尔值，成功删除文件返回 true，否则返回 false，文件不存在，不会抛出异常。当有多个线程删除文件并且您不想仅仅因为一个线程先这样做而抛出异常时，静默失败（不抛异常）很有用。

delete 可以删除空目录，但是如果目录不为空，则会抛出 DirectoryNotEmptyException 的异常。

---

## 文件的读写

### 小文件的读写

* readAllByte：读取所有信息到一个字节数组中。
* readString：读取所有信息到一个字符串中。
* readAllLines：读取所有行并返回一个字符串列表。
* write：将字节数组或者列表写入文件。
* writeString：将字符串写入文件。

读取信息，示例文件，`c:\User\dingj\Desktop\file.txt`：

```
123
Hello world
你好
```

读取信息，示例代码：

```java
Path p = Path.of("C:\\Users\\dingj\\Desktop\\file.txt");
byte[] bytes = Files.readAllBytes(p);
String s = Files.readString(p);
List<String> list = Files.readAllLines(p);

System.out.println("bytes: " + Arrays.toString(bytes));
System.out.println("---");
System.out.println("String: " + s);
System.out.println("---");
System.out.println("List<String>: " + list);
```

打印结果：

```
bytes: [49, 50, 51, 13, 10, 72, 101, 108, 108, 111, 32, 119, 111, 114, 108, 100, 13, 10, -28, -67, -96, -27, -91, -67]
---
String: 123
Hello world
你好
---
List<String>: [123, Hello world, 你好]
```

写入信息，示例代码：

```java
List<String> list = List.of("hello", "world", "123");

Path p = Path.of("C:\\Users\\dingj\\Desktop\\file.txt");
// 将列表写入文件，文件不存在则抛出异常
Files.write(p, list);
```

如果希望是追加模式，则需要添加 StandardOpenOption.APPEND 参数，另外如果一开始文件不存在，会抛出 NoSuchFileException，除了手动创建文件之外，也可以添加 StandardOpenOption.CREATE 参数。所以一般 APPEND 和 CREATE 组合使用：

```java
List<String> list = List.of("hello", "world", "123");
String s1 = "你好";
String s2 = "abc";

Path p = Path.of("C:\\Users\\dingj\\Desktop\\file.txt");
// 将列表写入文件，文件不存在，则创建文件，追加模式
Files.write(p, list, StandardOpenOption.CREATE, StandardOpenOption.APPEND);
// 将字节数组写入文件，追加模式
Files.write(p, s1.getBytes(StandardCharsets.UTF_8), StandardOpenOption.APPEND);
// 将字符串写入文件，追加模式
Files.writeString(p, s2, StandardCharsets.UTF_8, StandardOpenOption.APPEND);
```

### 大文件的读写

对于大文件，上面的方法就不适用了，过大的内容一次性塞到数组中，可能会造成OOM，Files 同样提供了多种方法，来读写大文件：

* newInputStream
* newOutputStream
* newBufferedReader
* newBufferedWriter

---

## 列出目录下的内容

一个目录下可能包含零个或者多个文件、子目录，可以通过 newDirectoryStream 方法获取一个实现了 DirectoryStream 接口的对象，DirectoryStream 是一个流，所以使用完需要关闭流，因为实现了 Closeable 接口，所以也可以写在 try-with-resource 语句中，另外还是实现了 Iterable 接口，所以可以使用加强 for 循环遍历。

下面的代码展示了如果列出某个文件夹下所有的文件：

```java
Path src = Path.of("C:\\Users\\dingj\\Desktop");
try (DirectoryStream<Path> ds = Files.newDirectoryStream(src)) {
    ds.forEach(System.out::println);
} catch (IOException e) {
    throw new RuntimeException(e);
}
```

### 使用通配符过滤目录内容

如果想要过滤内容，可以使用另外一个 newDirectoryStream，额外接受一个字符串，内容是 Glob 通配符，比如列出某个目录下与 Java 相关的文件：.class、.java 和 .jar 结尾的文件：

```java
Path src = Path.of("C:\\Users\\dingj\\Desktop");
try (DirectoryStream<Path> ds = Files.newDirectoryStream(src, "*.{java, class, jar}")) {
    ds.forEach(System.out::println);
} catch (IOException e) {
    throw new RuntimeException(e);
}
```

### 使用自己编写的过滤器来过滤目录内容

如果 Glob 无法满足需求，我们还可以通过定义一个 DirectoryStream.Filter 来做自定义的过滤器。

比如，定义一个过滤器，来过滤出目录类型的对象：

```java
DirectoryStream.Filter<Path> filter = Files::isDirectory;

Path src = Path.of("C:\\Users\\dingj\\Desktop");
try (DirectoryStream<Path> ds = Files.newDirectoryStream(src, filter)) {
    ds.forEach(System.out::println);
} catch (IOException e) {
    throw new RuntimeException(e);
}
```

综上，DirectoryStream 仅仅用于列出，遍历单个目录下的所有文件（深度为 1 层），如果想要遍历、查找所有目录下的文件（子目录的子目录的子目录之类的），就需要借助之后介绍的遍历目录树的机制，即 walkFileTree 方法。

---

## 遍历文件树

### FileVisitor

想要遍历一整棵文件树（而非像上一小节一样，只是获取单个目录下的一层文件），需要实现 FileVisitor 接口，此接口定义了在遍历树的过程中，访问每一个结点的必要操作：

1. 进入目录节点前的操作：在进入目录节点前调用。
2. 退出目录节点后的操作：遍历完所有目录下的文件节点后调用。
3. 访问目录下的文件的操作：访问文件时调用，文件的 BasicFileAttribute 会传入该方法。
4. 访问目录下的文件失败的操作：如果文件无法访问则调用，你可以选择抛出异常，打印在控制台或者记录在日志文件中。

如果你不想从头实现 FileVisitor 接口，你可以选择 SimpleFileVisitor，它提供了 FileVisitor 的基本实现。

下面是一个实现了 SimpleFileVisitor 的示例：

```java
class PrintFiles extends SimpleFileVisitor<Path> {
    // 进入目录前的操作
    @Override
    public FileVisitResult preVisitDirectory(Path dir, BasicFileAttributes attrs) {
        System.out.format("开始访问目录: %s\n", dir);
        // 这里有个目录下面的层级非常多，我不希望遍历它，可以选择排除掉
        if (Objects.equals(dir.getFileName().toString(), "StegaStamp")) {
            // 跳过这棵子树
            return FileVisitResult.SKIP_SUBTREE;
        }
        return FileVisitResult.CONTINUE;
    }

    // 访问目录下的文件时的操作
    @Override
    public FileVisitResult visitFile(Path file, BasicFileAttributes attrs) {
        System.out.format("访问文件: %s, 创建时间: %s, 大小: %s 字节\n", file, attrs.creationTime(), attrs.size());
        return FileVisitResult.CONTINUE;
    }

    // 访问目录下的文件失败时的操作
    @Override
    public FileVisitResult visitFileFailed(Path file, IOException exc) {
        System.err.format("访问文件 %s 时出错: %s\n", file, exc);
        return FileVisitResult.CONTINUE;
    }

    // 访问目录下所有的节点结束后，退出目录时的操作
    @Override
    public FileVisitResult postVisitDirectory(Path dir, IOException exc) {
        System.out.format("目录: %s 访问结束\n", dir);
        return FileVisitResult.CONTINUE;
    }
}

```

一旦实现了 FileVisitor，就可以通过 walkFileTree 来遍历目录：

```java
Path src = Path.of("C:\\Users\\dingj\\Desktop");
try {
    Files.walkFileTree(src, new PrintFiles());
} catch (IOException e) {
    throw new RuntimeException(e);
}
```

文件树的遍历属于深度优先遍历，但是它无法保证遍历的顺序，同一目录下的文件遍历顺序可能是不确定的。如果你需要对文件树做出修改的操作，则需要仔细考虑 FileVisitor 的实现方法。

比方说，在编写递归删除的时候，得先将目录下的文件都删掉，再删除目录本身，在这种情况下，应该把删除目录的逻辑代码放在 postVisitDirectory 中。

而如果想要编写递归复制，则需要先创建目录，再将文件们拷贝过去，此时应该将创建目录的逻辑代码放在 preVisitDirectory 中。

如果正在编写文档搜索，则可以在 visitFile 方法中执行比较。此方法查找与您的条件匹配的所有文档，但这种方式无法查找目录。如果要同时查找文档和目录，还必须在 preVisitDirectory 或 postVisitDirectory 方法中执行比较。

### 遍历时的控制流

加入你在编写一个查找目录下某个文件的代码，你希望找到第一个符合条件的文件就立刻退出，又或者向上面一小节的例子中，碰到某个子目录希望跳过，不进行遍历搜索，这时候就需要借助遍历文件树的控制流——FileVisitResult。

FileVisitor 的四个方法都返回 FileVisitResult 对象，它是一个枚举类，有以下几个值：

1. CONTINUE: 表示文档遍历应继续。如果 preVisitDirectory 方法返回 CONTINUE，则继续访问该目录。
2. TERMINATE: 立即中止文档遍历。返回此值后，不会调用进一步的文档遍历方法。
3. SKIP_SUBTREE: 当 preVisitDirectory 返回此值时，将跳过指定的目录及其子目录。相当于树的“剪枝”操作。
4. SKIP_SIBLING: 当 preVisitDirectory 返回此值时，不会访问指定的目录，不会调用 postVisitDirectory，也不会再访问未访问的同级文件。如果从 postVisitDirectory 方法返回，则不会再访问兄弟节点。本质上，指定目录中不会发生任何进一步的事情。

比如，下面的代码，在碰到名为 “test” 的目录名时，不进行访问：

```java
public FileVisitResult preVisitDirectory(Path dir, BasicFileAttributes attrs) {
    if (dir.getFileName().toString().equals("test")) {
         return FileVisitResult.SKIP_SUBTREE;
    }
    return FileVisitResult.CONTINUE;
}
```

下面的代码，在找到指定的文件后，立刻打印一段信息，随后终止并退出文件树的遍历：

```java
// 我们需要寻找的文件
Path lookingFor = ...;

public FileVisitResult visitFile(Path file, BasicFileAttributes attr) {
    if (file.getFileName().equals(lookingFor)) {
        System.out.println("Located file: " + file);
        return FileVisitResult.TERMINATE;
    }
    return FileVisitResult.CONTINUE;
}
```

---

## 观测目录中的变化
