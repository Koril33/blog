---
title: "Java基础之注解"
date: 2023-09-09T11:13:55+08:00
tags: []
featured_image: "images/background.jpg"
summary: "注解的意义和用法"
toc: true
---

## 前言

注解（Annotation），是一种元数据，所谓元数据，是关于数据的数据，关于信息的信息。

假设有一段数据：

```
今天，阳光明媚，我很开心。
```

这一段完整的数据，我们可以通过一些方式去描述它，比如，它包含三个标点符号；它属于句子；它由 10 个汉字组成。这些描述了这段信息的信息就叫做元数据。

```
# 标点符号: 3 个
# 汉字: 10 个
# 类型: 语句
今天，阳光明媚，我很开心。
```

在平时看书的时候，很多人也喜欢对原始文章内容的一些字词，画圈做标记，然后在旁边批注上自己搜集到的额外的信息，或者是自己的所思所想，对标记的内容进行补充和注释。

在编程中，对于一个源代码文件而言，它的原始内容就是一行行的代码，而描述补充这些代码的内容就是元数据，在 Java 中，单独使用注解来作为元数据的标记。

---

## 注解的特点

注解不会对代码的执行逻辑和流程构成直接影响，它更多的是对信息的标记。

一般有以下一些用法：

* 向编译器提供信息：注解可以帮助编译器提供一些信息去检测错误或者压制警告。
* 向软件工具提供信息：可以在编译器编译时和部署时进行额外处理，比如根据注解生成一些代码，文档，等等。
* 运行时处理：一些注解可以在运行时起作用。

---

## 注解的基础

### 注解的格式

最简单的注解形式如下：

```java
@Entity
```

使用 @ 符号向编译器表明跟随在这个符号后面的是一个注解，下面这个例子中，注解的名称是 Override：

```java
@Override
void mySuperMethod() { ... }
```

注解包含元素（elements），元素会被赋予一些值：

```java
@Author(
    name = "Koril",
    date = "2023-09-10"
)
class MyClass { ... }
```

或者是：

```java
@SuppressWarning(value = "unchecked")
void myMethod() { ... }
```

如果注解中仅仅包含一个元素，并且该元素的名称叫做 value，那么可以省略元素名称：

```java
@SuppressWarning("unchecked")
void myMethod() { ... }
```

如果注解一个元素也没有，那么括号也可以被省略掉，比如常用的 @Override 注解。

也可以在同一个声明上使用多个注解：

```java
@Author(name = "koril")
@EBook
class MyClass { ... }
```

如果多个注解属于一个类型，它们则被称为重复性注解：

```java
@Author(name = "Alice")
@Author(name = "Bob")
class MyClass { ... }
```

Java 平台提供了一些官方注解，定义在`java.lang`包和`java.lang.annotation`包中。在之前的实例中，Override 和 SuppressWarning 都属于官方注解。当然也可以定义自己的注解，前面示例中的 Author  和 EBook 就属于自定义注解类型。

### 声明注解

注解可以取代掉很多代码中的注释信息。

举个例子，很多软件团队会在类声明的起始行，添加一些额外的信息：

```java
public class Generation3List extends Generation2List {

   // Author: John Doe
   // Date: 3/17/2002
   // Current revision: 6
   // Last modified: 4/12/2004
   // By: Jane Doe
   // Reviewers: Alice, Bill, Cindy

   // class code goes here

}
```

为了用注解替代掉这些注释信息，必须先声明一个注解：

```java
@interface ClassPreamble {
   String author();
   String date();
   int currentRevision() default 1;
   String lastModified() default "N/A";
   String lastModifiedBy() default "N/A";
   // Note use of array
   String[] reviewers();
}
```

注解的声明非常类似于接口（interface）的声明方式，使用 @interface 标记声明的注解。声明体中包含了一些注解的元素声明，看来很像是接口中的方法声明，但不同的是，它们可以包含一些可选的默认值。

定义完了注解，就可以用它来去掉刚刚那些注释信息：

```java
@ClassPreamble (
   author = "John Doe",
   date = "3/17/2002",
   currentRevision = 6,
   lastModified = "4/12/2004",
   lastModifiedBy = "Jane Doe",
   // Note array notation
   reviewers = {"Alice", "Bob", "Cindy"}
)
public class Generation3List extends Generation2List {

// class code goes here

}
```

