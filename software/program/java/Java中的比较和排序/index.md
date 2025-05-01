---
title: "Java中的比较和排序"
date: 2021-05-27T12:04:15+08:00
summary: "比较和排序是所有应用的基石"
featured_image: "images/background.jpg"
toc: true
---

## 前言

在编写应用的过程当中，最常见的就是操作一系列的数据容器来实现对数据的修改及传输。一系列的数据容器指的是 Java 中的 Array 和 Collection，而操作则泛指对集合的增、删、改、查。

本文简单讨论了关于集合的其中一项最为重要的部分——排序。

排序是建立在比较之上的，对于普通类型而言，可以使用默认的比较策略，通过对比数值的大小，或者是字符编码的前后顺序来排序。但是，自己定义的类型，往往会产生一些较为复杂的比较策略。比如，先对某个成员变量升序排列，相同的话，则对另外一个成员变量按降序排列。

Java 的排序对不同类型的元素采取了不同的算法策略：

1. 如果容器是存储了基本类型（Primitive Type）的数组，底层采用“双枢轴快速排序（DualPivotQuicksort）”。

2. 如果容器存储的是非自然排序的引用类型元素（即未实现`Comparable`接口），底层采用 “TimSort”。

3. 如果容器存储的是自然排序的引用类型元素（实现了`Comparable`接口），底层采用“ComparableTimSort”。

基本类型的元素本身就是可排序的，而引用类型则需要自定义排序规则，这个自定义要么实现`Comparable` 接口，要么给排序的算法函数传递一个实现了`Comparator`接口的比较器。

接下来，会依次介绍基本类型排序，自然排序，比较器，多条件排序，lambda 语法糖等各个概念。

---

## 基本类型排序

Java 有八大基本类型：byte、short、int、long、float、double、boolean、char。

对于基本类型的容器，Java 只有数组这一个选择，因为自动拆装箱的存在和泛型的实现机制，集合存储的都是引用类型（存储基本类型的时候，自动装箱成包装类）。所以在对基本类型排序时，用法非常简单和透明，将数组作为参数，传递给排序算法，排序算法在数组上进行原地修改排序。

### 基本类型按升序排序

具体的实现放在了`java.util.Arrays`中，此类包含了许多用于操作数组的方法，包括了排序（sort），搜索（binarySearch），比较（equals、compare、mismatch），拷贝（copyOf、copyOfRange），填充（fill），哈希（hashCode），转换方法（asList、stream）等等。

尽管这个工具类有将近 9k 行代码，但是大部分都是上述功能的重载方法，`Arrays.sort()`就是其中之一，它对基本类型的每一个（除了`boolean`之外）都进行了重载：

```java
public static void sort(int[] a) {
    DualPivotQuicksort.sort(a, 0, a.length - 1, null, 0, 0);
}

public static void sort(int[] a, int fromIndex, int toIndex) {
    rangeCheck(a.length, fromIndex, toIndex);
    DualPivotQuicksort.sort(a, fromIndex, toIndex - 1, null, 0, 0);
}


public static void sort(long[] a) {
    DualPivotQuicksort.sort(a, 0, a.length - 1, null, 0, 0);
}

public static void sort(long[] a, int fromIndex, int toIndex) {
    rangeCheck(a.length, fromIndex, toIndex);
    DualPivotQuicksort.sort(a, fromIndex, toIndex - 1, null, 0, 0);
}


public static void sort(short[] a) {
    DualPivotQuicksort.sort(a, 0, a.length - 1, null, 0, 0);
}

public static void sort(short[] a, int fromIndex, int toIndex) {
    rangeCheck(a.length, fromIndex, toIndex);
    DualPivotQuicksort.sort(a, fromIndex, toIndex - 1, null, 0, 0);
}


public static void sort(char[] a) {
    DualPivotQuicksort.sort(a, 0, a.length - 1, null, 0, 0);
}

public static void sort(char[] a, int fromIndex, int toIndex) {
    rangeCheck(a.length, fromIndex, toIndex);
    DualPivotQuicksort.sort(a, fromIndex, toIndex - 1, null, 0, 0);
}


public static void sort(byte[] a) {
    DualPivotQuicksort.sort(a, 0, a.length - 1);
}

public static void sort(byte[] a, int fromIndex, int toIndex) {
    rangeCheck(a.length, fromIndex, toIndex);
    DualPivotQuicksort.sort(a, fromIndex, toIndex - 1);
}


public static void sort(float[] a) {
    DualPivotQuicksort.sort(a, 0, a.length - 1, null, 0, 0);
}

public static void sort(float[] a, int fromIndex, int toIndex) {
    rangeCheck(a.length, fromIndex, toIndex);
    DualPivotQuicksort.sort(a, fromIndex, toIndex - 1, null, 0, 0);
}


public static void sort(double[] a) {
    DualPivotQuicksort.sort(a, 0, a.length - 1, null, 0, 0);
}

public static void sort(double[] a, int fromIndex, int toIndex) {
    rangeCheck(a.length, fromIndex, toIndex);
    DualPivotQuicksort.sort(a, fromIndex, toIndex - 1, null, 0, 0);
}
```

Java 不支持对形参指定默认值，所以另外一部分的重载是针对`fromIndex`和`toIndex`这两个参数，它俩指定了数组排序的范围，如果不指定范围，则默认对整个数组进行排序。

因为这几个基本类型都是数值直接比较，所以相对于后面要引入的对象比较器而言比较简单，默认是升序排列，以下是示例：

```java
int[] a = {84, 57, 33, 37, 9, 7, 63, 86, 14, 13};
Arrays.sort(a);
结果：
[7, 9, 13, 14, 33, 37, 57, 63, 84, 86]

int[] a = {84, 57, 33, 37, 9, 7, 63, 86, 14, 13};
// 指定范围[3, 8)
Arrays.sort(a, 3, 8);
结果：
[84, 57, 33, 7, 9, 37, 63, 86, 14, 13]
```

### 基本类型按降序排序

对于基本类型，如果想要降序排列，Java 并没有直接的 API，需要自己编写代码，思路大概就是先升序，再反转。可以直接操纵数组，也可以使用 Java 8 的 Stream 实现，或者是借助第三方库，这里简单实现以下前两种方法：

```java
// 传统方法，对已经升序排列的数组，直接对调前后数组元素，实现降序
public static void reverse(int[] a, int fromIndex, int toIndex) {
    int i = fromIndex, j = toIndex - 1;
    while (i < j) {
        swap(a, i++, j--);
    }
}
// 只传数组，则默认对调整个数组
public static void reverse(int[] a) {
    reverse(a, 0, a.length - 1);
}
// 交换数组元素的方法
private static void swap(int[] a, int i, int j) {
    int tmp = a[i];
    a[i] = a[j];
    a[j] = tmp;
}

// 使用 Stream 流处理数组降序排列
public static int[] descSortByStream(int[] a) {
    return Arrays.stream(a)
            // 从 IntStream 变成 Stream<Integer>
            .boxed()
            // 传入一个比较器，该比较器与自然排序相反
            .sorted(Comparator.reverseOrder())
            // 从 Stream<Integer> 再变回 IntStream
            .mapToInt(Integer::intValue)
            // 从 IntStream 变成数组
            .toArray();
}


// 使用 Guava 第三方库，结合 Collections.sort()
Collections.sort(Ints.asList(a), Comparator.reverseOrder());
// 或者用 List.sort()
Ints.asList(a).sort(Comparator.reverseOrder());
```

---

## 引用类型排序

### 引用类型的比较

引用类型就是除了基本类型之外的，平时使用的各种对象，对于这些对象而言，有些是可以比较的，有些是不能够比较的。对于希望实现比较功能的类，我们在创造这个类的时候必须赋予它自然排序（natural ordering）的能力。

Java 中实现对象间的比较方式主要是实现 Comparable 接口或 Comparator 接口。这两个接口存在语义上的差别，Comparable 强调的是某个类型是可比较的；而 Comparator 是一个比较器，放入两个对象，返回比较的结果。就像你可以说扔掉的可乐瓶是可回收的（Recyclable）东西，但不能说它是一个可回收器（ Recycler）。

### 自然排序

一个类拥有自然排序指的是，这个类天生就自带排序规则（由创建它的程序员制定），后面使用这个类的程序员不需要关心它具体的排序规则是什么，以及此规则底层是如何实现的。他只需要知道能把这个类生成的对象们丢到数组或者集合中，肯定可以调用`Arrays.sort()`或者`Collections.sort()`来进行排序处理。

想要给创造的类赋予这个规则，只需要实现`Comparable`接口，并且实现其中的唯一一个方法，我们把实现`Comparable`接口的类称为具有自然排序的类，把`compareTo`方法称为自然比较方法：

```java
package java.lang;
import java.util.*;

public interface Comparable<T> {
    public int compareTo(T o);
}
```

平常最常使用的那些类都实现了这个接口，比如：`String`、基本类型的包装器类、时间类等等。它们都实现了`compareTo`方法，所以我们可以在集合中很方便的对它们的对象们进行排序：

```java
// 对字符串列表排序
List<String> names = Arrays.asList("Bob", "Mike", "Frank", "Alice");

Collections.sort(names);
System.out.println(names);

// 对日期列表排序
List<LocalDate> dates = Arrays.asList(
        LocalDate.of(2022, 1, 1),
        LocalDate.of(2023, 2, 5),
        LocalDate.of(2021, 12, 31)
);
Collections.sort(dates);
System.out.println(dates);

// 对浮点数列表排序
List<Double> doubles = Arrays.asList(3.11, 2.0, 1.8, 5.4, 8.2);
Collections.sort(doubles);
System.out.println(doubles);


结果：
[Alice, Bob, Frank, Mike]
[2021-12-31, 2022-01-01, 2023-02-05]
[1.8, 2.0, 3.11, 5.4, 8.2]
```

`String` 实现的 `compareTo`：

```java
public int compareTo(String anotherString) {
    byte v1[] = value;
    byte v2[] = anotherString.value;
    if (coder() == anotherString.coder()) {
        return isLatin1() ? StringLatin1.compareTo(v1, v2)
                          : StringUTF16.compareTo(v1, v2);
    }
    return isLatin1() ? StringLatin1.compareToUTF16(v1, v2)
                      : StringUTF16.compareToLatin1(v1, v2);
}

@HotSpotIntrinsicCandidate
public static int compareTo(byte[] value, byte[] other) {
    int len1 = value.length;
    int len2 = other.length;
    return compareTo(value, other, len1, len2);
}

public static int compareTo(byte[] value, byte[] other, int len1, int len2) {
    int lim = Math.min(len1, len2);
    for (int k = 0; k < lim; k++) {
        if (value[k] != other[k]) {
            // 根据每一个 char 来比较，顺序就和 Unicode 顺序一样
            return getChar(value, k) - getChar(other, k);
        }
    }
    return len1 - len2;
}
```

`LocalDate`实现的 `compareTo`：

```java
@Override  // override for Javadoc and performance
public int compareTo(ChronoLocalDate other) {
    if (other instanceof LocalDate) {
        return compareTo0((LocalDate) other);
    }
    return ChronoLocalDate.super.compareTo(other);
}

int compareTo0(LocalDate otherDate) {
    // 先比较年，再比较月，最后比较天数
    int cmp = (year - otherDate.year);
    if (cmp == 0) {
        cmp = (month - otherDate.month);
        if (cmp == 0) {
            cmp = (day - otherDate.day);
        }
    }
    return cmp;
}
```

`Double`实现的`compareTo`：

```java
public int compareTo(Double anotherDouble) {
    return Double.compare(value, anotherDouble.value);
}

public static int compare(double d1, double d2) {
    if (d1 < d2)
        return -1;           // Neither val is NaN, thisVal is smaller
    if (d1 > d2)
        return 1;            // Neither val is NaN, thisVal is larger

    // Cannot use doubleToRawLongBits because of possibility of NaNs.
    long thisBits    = Double.doubleToLongBits(d1);
    long anotherBits = Double.doubleToLongBits(d2);

    return (thisBits == anotherBits ?  0 : // Values are equal
            (thisBits < anotherBits ? -1 : // (-0.0, 0.0) or (!NaN, NaN)
             1));                          // (0.0, -0.0) or (NaN, !NaN)
}
```

`Integer`实现的`compareTo`：

```java
public int compareTo(Integer anotherInteger) {
    return compare(this.value, anotherInteger.value);
}

public static int compare(int x, int y) {
    return (x < y) ? -1 : ((x == y) ? 0 : 1);
}
```

可以看出，`Number`类型的`compareTo`都很直观，比较值的大小即可。但其他的类，就需要人为的定义规则了，比如日期是按照年、月、日顺序依次排序，字符串是按照字符（`String`的底层是数组，JDK1.8 以及之前都是 char 数组，JDK1.9之后是 byte 数组）逐个比较。

### 继承 Comparable 接口定义排序规则

上面简单了解了下 Java 中的常用类是如何实现`Comparable`接口的，接下来就自己尝试实现一个带有自然排序规则的类。

先从最简单的开始，`Person`类拥有两个字段，一个是`String`类型的 name，一个是`int`类型的 age：

```java
// Lombok 注解，省去 getter/setter 模板代码
@Data
@AllArgsConstructor
public class Person {
    // 姓名
    private String name;
    // 年龄
    private int age;
}
```

创建一个人群列表：

```java
List<Person> people = Arrays.asList(
        new Person("Bob", 23),
        new Person("Mike", 14),
        new Person("Alice", 22),
        new Person("John", 23),
        new Person("Jack", 32),
        new Person("Tom", 23)
);
```

然后尝试使用`Collections.sort()`来对人群列表做个排序：

```java
Collections.sort(people);

结果：
java: 对于sort(java.util.List<cn.korilweb.entity.Person>), 找不到合适的方法
    方法 java.util.Collections.<T>sort(java.util.List<T>)不适用
      (推论变量 T 具有不兼容的上限
        等式约束条件：cn.korilweb.entity.Person
        下限：java.lang.Comparable<? super T>)
    方法 java.util.Collections.<T>sort(java.util.List<T>,java.util.Comparator<? super T>)不适用
      (无法推断类型变量 T
        (实际参数列表和形式参数列表长度不同))
```

报错的原因是由于我们违反了规则，`Collections.sort()`并不知道`Person`这个自定义的类该以什么规则去排序，换句话说，我们还没有实现`Comparable`接口去定义`Person`的自然排序规则。

```java
@Data
@AllArgsConstructor
public class Person implements Comparable<Person> {

    private String name;
    private int age;

    @Override
    public int compareTo(Person p) {
        return Integer.compare(age, p.age);
    }
}

Collections.sort(people);

结果：
[Person(name=Mike, age=14), Person(name=Alice, age=22), Person(name=Bob, age=23), Person(name=John, age=23), Person(name=Tom, age=23), Person(name=Jack, age=32)]
```

people 列表成功的按照`compareTo`中给定的规则，依年龄从小到大升序排列`Person`对象。如果我们希望按照年龄降序排列，也很简答，交换一下比较的值即可：

```java
@Override
public int compareTo(Person p) {
    // 交换一下比较的顺序
    return Integer.compare(p.age, age);
}
```

我们点开`Collections`的源码可以看到，有两个重载的`sort`方法，一个只接受`List<T>`，另外一个接受一个`List<T>`以及一个`Comparator<? super T>`比较器：

```java
// 只接受列表
public static <T extends Comparable<? super T>> void sort(List<T> list) {
    list.sort(null);
}
// 除了列表以外还接受一个比较器
public static <T> void sort(List<T> list, Comparator<? super T> c) {
    list.sort(c);
}
```

看得出来，重载是为了让比较器可以默认为 null，然后调用`List.sort()`，`List.sort()`是一个默认接口方法（default 方法，JDK1.8 的新特性，可以在接口里实现一个默认方法，实现接口的类可以直接调用），所以上面的代码还可以写成：

```java
Collections.sort(people);
// 可以写成
people.sort(null);
```

下面继续看看`List.sort()`是如何实现排序的：

```java
default void sort(Comparator<? super E> c) {
    // 先将列表转成数组
    Object[] a = this.toArray();
    // 用 Arrays.sort 对数组排序
    Arrays.sort(a, (Comparator) c);
    ListIterator<E> i = this.listIterator();
    for (Object e : a) {
        i.next();
        i.set((E) e);
    }
}
```

众所周知，`List`接口最出名的两实现类是`ArrayList`和`LinkedList`，`LinkedList`没有覆写这个方法，所以`LinkedList`对象直接调用`List.sort`，先，而`ArrayList`则覆写了此方法：

```java
@Override
@SuppressWarnings("unchecked")
public void sort(Comparator<? super E> c) {
    final int expectedModCount = modCount;
    // ArrayList 底层就是维护了一个数组，将此数组传给 Arrays.sort 排序
    Arrays.sort((E[]) elementData, 0, size, c);
    if (modCount != expectedModCount)
        throw new ConcurrentModificationException();
    modCount++;
}
```

不管是覆写还是直接继承，我们看到`List.sort`的方法内部还是调用了一开始就讲过的`Arrays.sort()`，兜了一圈，又绕回到这个工具类了。`Array.sort()`对于对象排序总共有四个重载方法：

```java
// 接受一个数组和一个比较器，默认 fromIndex=0，toIndex=a.length
public static <T> void sort(T[] a, Comparator<? super T> c) {
    if (c == null) {
        // 没有比较器，就调用 ComparableTimSort.sort
        sort(a);
    } else {
        if (LegacyMergeSort.userRequested)
            legacyMergeSort(a, c);
        else
            // 有比较器，就调用 TimSort.sort
            TimSort.sort(a, 0, a.length, c, null, 0, 0);
    }
}

// 接受一个数组和一个比较器，指定 fromIndex 和 toIndex
public static <T> void sort(T[] a, int fromIndex, int toIndex,
                                Comparator<? super T> c) {
    // 没有比较器，就调用 ComparableTimSort.sort
    if (c == null) {
        sort(a, fromIndex, toIndex);
    } else {
        rangeCheck(a.length, fromIndex, toIndex);
        if (LegacyMergeSort.userRequested)
            legacyMergeSort(a, fromIndex, toIndex, c);
        else
            // 有比较器，就调用 TimSort.sort
            TimSort.sort(a, fromIndex, toIndex, c, null, 0, 0);
    }
}

// 接受一个数组，默认 fromIndex=0，toIndex=a.length
public static void sort(Object[] a) {
    if (LegacyMergeSort.userRequested)
        legacyMergeSort(a);
    else
        ComparableTimSort.sort(a, 0, a.length, null, 0, 0);
}

// 接受一个数组，指定 fromIndex 和 toIndex
public static void sort(Object[] a, int fromIndex, int toIndex) {
    rangeCheck(a.length, fromIndex, toIndex);
    if (LegacyMergeSort.userRequested)
        legacyMergeSort(a, fromIndex, toIndex);
    else
        ComparableTimSort.sort(a, fromIndex, toIndex, null, 0, 0);
}
```

上面的第一个和第二个方法是一组，第三个和第四个方法是一组，如果比较器不为 null，调用`TimSort.sort()`，比较器为 null，则调用 `ComparableTimSort.sort()`。

现在，我们已经看到了三个排序算法类，它们分别服务于不同的模型。

1. DualPivotQuicksort：处理基本类型的排序。
2. ComparableTimSort：处理没有传入比较器，但实现了`Comparable`接口的类。
3. TimSort：处理额外传入比较器的类。

关于为什么需要分别采用不同的算法来对基本类型和引用类型进行排序的问题，可以参考：

1. https://stackoverflow.com/questions/3707190/why-does-javas-arrays-sort-method-use-two-different-sorting-algorithms-for-diff
2. https://stackoverflow.com/questions/4305004/why-arrays-sort-is-quicksort-algorithm-why-not-another-sort-algorithm

### 传入比较器定义排序规则

我们此时的`Person`类，规则是写死在了`compareTo`中，如果在使用该类的时候，我们想要换个规则来进行排序，除了修改`compareTo`方法之外，我们还可以选择传入一个自定义的比较器。比较器`Comparator`也是一个接口，如果我们想要自定义一个比较器，就必须实现它的`compare`方法：

```java
package java.util;

@FunctionalInterface
public interface Comparator<T> {
    int compare(T o1, T o2);
}
```

现在，定义一个比较器，来实现按照名字，降序排列的功能：

```java
public class DescNameComparator implements Comparator<Person> {
    @Override
    public int compare(Person p1, Person p2) {
        String n1 = p1.getName();
        String n2 = p2.getName();
        // 按照名字降序排列，调用 String 的 compareTo 方法
        // 如果升序，则是：return n1.compareTo(n2);
        return n2.compareTo(n1);
    }
}
```

然后将这个比较器的实例对象，传入`Collections.sort()`或者`List.sort()`中：

```java
Collections.sort(people, new DescNameComparator());
// 或者
people.sort(new DescNameComparator());

结果：
[Person(name=Tom, age=23), Person(name=Mike, age=14), Person(name=John, age=23), Person(name=Jack, age=32), Person(name=Bob, age=23), Person(name=Alice, age=22)]
```

可以看到，这回没有按照年龄排序，而是按照姓名降序排列。

这回，由于传入了比较器，走的是 TimSort 类排序：

```java
public static <T> void sort(T[] a, Comparator<? super T> c) {
    // 传入了比较器，c 不为 null
    if (c == null) {
        sort(a);
    } else {
        if (LegacyMergeSort.userRequested)
            legacyMergeSort(a, c);
        else
            // 调用 TimSort 类排序
            TimSort.sort(a, 0, a.length, c, null, 0, 0);
    }
}
```

---

## 多条件排序

上一小节中，针对于`Person`类分别按照姓名和年龄做了排序，但是在实际场景中我们往往需要多条件的排序，比如：先按字段1升序排列，再按字段2降序排列，最后按字段3升序排列。

### 自然顺序下的多条件排序

我们按照`Person`类，先按照年龄降序排列，再按照姓名升序排列作为例子。

如果是自然排序，我们需要修改`compareTo()`的实现：

```java
@Override
public int compareTo(Person p) {
    // 先按照年龄降序排列
    int c = Integer.compare(p.age, age);
    
    // 如果要求年龄升序排列，交换前后顺序即可
    // int c = Integer.compare(age, p.age);

    if (c == 0) {
        // 再按照姓名升序排列
        c = name.compareTo(p.getName());

        // 如果要求姓名也降序，交换前后顺序即可
        // c = p.getName().compareTo(name);
    }
    return c;
}
```

也可以写成下面这种形式：

```java
@Override
public int compareTo(Person p) {
    // 先按照年龄降序排列
    if (age != p.age) {
        return p.age - age;
        // 如果要求年龄升序排列，交换前后顺序即可
        // return age - p.age;
    }

    // 再按照姓名升序排列
    return name.compareTo(p.getName());

    // 如果要求姓名也降序，交换前后顺序即可
    // return p.getName().compareTo(name);
}
```

### 比较器下的多条件排序

当然，传入一个自定义的比较器也是可以实现相同的功能的：

```java
// 自定义一个多条件比较器
class multiFieldComparator implements Comparator<Person> {
    @Override
    public int compare(Person p1, Person p2) {
        // 先比较年龄，降序排列
        int c = Integer.compare(p2.getAge(), p1.getAge());
        if (c == 0) {
            // 再比较姓名，升序排列
            c = p1.getName().compareTo(p2.getName());
        }
        return c;
    }
}

// 排序的时候，传入自定义比较器的实例
Collections.sort(people, new multiFieldComparator());
```

不管是，自然排序还是传入自定义的比较器，结果都是一致的：

```java
Person(name=Jack, age=32)
Person(name=Bob, age=23)
Person(name=John, age=23)
Person(name=Tom, age=23)
Person(name=Alice, age=22)
Person(name=Mike, age=14)
```

### 复杂性的增加

上面是两个字段的多条件排序，一旦排序的字段增加，代码的复杂性随之增高，可读性随之下降。考虑以下的需求，有一个`Student`类，除了姓名和年龄外，还有几条成绩分数的字段，现在的要求是：

1. 先按照数学成绩降序排列

2. 再按照英语成绩降序排列

3. 再按照语文成绩升序排列

4. 最后按照科学成绩升序排列。

```java
@Data
@AllArgsConstructor
public class Student {
    // 姓名和年龄
    private String name;
    private int age;
    // 各科成绩
    private int math;
    private int english;
    private int chinese;
    private int science;
}
```

如果是自然排序，我们需要修改`Student.compareTo()`：

```java
@Override
public int compareTo(Student s) {
    // 数学，降序
    int c = Integer.compare(s.math, math);
    if (c == 0) {
        // 英语，降序
        c = Integer.compare(s.english, english);
        if (c == 0) {
            // 语文，升序
            c = Integer.compare(chinese, s.chinese);
            if (c == 0) {
                // 科学，升序
                c = Integer.compare(science, s.science);
            }
        }
    }
    return c;
}
```

或者这样：

```java
@Override
public int compareTo(Student s) {
    // 数学，降序
    int c = Integer.compare(s.math, math);
    if (c == 0) {
        // 英语，降序
        c = Integer.compare(s.english, english);
    }
    if (c == 0) {
        // 语文，升序
        c = Integer.compare(chinese, s.chinese);
    }
    if (c == 0) {
        // 科学，升序
        c = Integer.compare(science, s.science);
    }
    return c;
}
```

亦或者这样：

```java
@Override
public int compareTo(Student s) {
    // 数学，降序
    if (s.math != math) {
        return s.math - this.math;
    }
    // 英语，降序
    if (s.english != this.english) {
        return s.english - english;
    }
    // 语文，升序
    if (chinese != s.chinese) {
        return chinese - s.chinese;
    }
    // 科学，升序
    return science - s.science;
}
```

使用比较器的代码逻辑，大同小异，这里不在赘述，但无论如何，这种形式远远称不上”拥有良好的可读性“，在编写的过程中，也必须谨小慎微。必须清楚的知道，对哪些字段排序，考虑升序和降序的时候，要注意前后变量顺序不能弄反了，最后，还要兼顾不同的程序员，对于多字段排序的实现有不同的写法，比如上面的三种，很难一下子就能从源码判断出到底排序的需求是什么。

### 函数式和比较器链的救场

Java 8 提供了函数式编程的功能：方法引用、lambda 表达式、函数式接口。凭借着这些工具我们可以轻松的实现可读性高的比较器链，就像 SQL 语法一样，容易阅读，假如`Student`类是数据库中的一张表的话，完成上面的功能，用 SQL 语法可以这么写：

```sql
SELECT
    *
FROM 
    student
WHERE
    (条件语句)
ORDER BY 
    math DESC,
    english DESC,
    chinese ASC,
    science ASC
```

SQL 的这个多条件排序，可读性很高，可以一眼看出需要对哪些字段排序，每个字段是升序还是降序。

SQL 语言是一门 DSL（Domain Specific Language），即领域特定语言，Java 8 通过引入函数式编程的特性，也可以实现这样类似的比较器链的效果：

```java
Comparator<Student> comparator =
        Comparator
                .comparing(Student::getMath, Comparator.reverseOrder())
                .thenComparing(Student::getEnglish, Comparator.reverseOrder())
                .thenComparing(Student::getChinese, Comparator.naturalOrder())
                .thenComparing(Student::getScience, Comparator.naturalOrder());
Collections.sort(students, comparator);
```

总共 4 个字段，通过`comparing`和`thenComparing`两个方法串联成了比较器链，最后将生成的比较器传递给`Collections.sort`即可。

#### 继续省略重复代码

由于语文和科学是升序排列，这恰好就是`Integer`的自然排序规则，所以`Comparator.naturalOrder()`和 SQL 中的`ASC`一样是可以省略不写的。

除此之外，Java 还引入`import static`机制，通过引入`Comparator`的静态方法，可以省下不少重复的代码：

```java
import static java.util.Comparator.comparing;
import static java.util.Comparator.reverseOrder;

Comparator<Student> comparator =
            comparing(Student::getMath, reverseOrder())
            .thenComparing(Student::getEnglish, reverseOrder())
            .thenComparing(Student::getChinese)
            .thenComparing(Student::getScience);
Collections.sort(students, comparator);
```

接下来的内容主要涉及逆序比较器和自然排序比较器的源码，这些代码主要位于Comparator 接口、Comparators 类、Collections 类之中。

#### ReverseComparator

点开`reverseOrder()`的源码，可以看到：

```java
// 该方法位于 java.util.Comparator 中
public static <T extends Comparable<? super T>> Comparator<T> reverseOrder() {
    return Collections.reverseOrder();
}

// 该方法位于 java.util.Collections 中
public static <T> Comparator<T> reverseOrder() {
    return (Comparator<T>) ReverseComparator.REVERSE_ORDER;
}

// 该类位于 java.util.Collections 中
private static class ReverseComparator
    implements Comparator<Comparable<Object>>, Serializable {

    private static final long serialVersionUID = 7207038068494060240L;

    static final ReverseComparator REVERSE_ORDER
        = new ReverseComparator();

    public int compare(Comparable<Object> c1, Comparable<Object> c2) {
        return c2.compareTo(c1);
    }

    private Object readResolve() { return Collections.reverseOrder(); }

    @Override
    public Comparator<Comparable<Object>> reversed() {
        return Comparator.naturalOrder();
    }
}
```

通过`return c2.compareTo(c1)`可以看到，该方法的作用返回一个与自然排序相反（降序）的比较器。

#### NaturalOrderComparator

点开`naturalOrder`的源码：

```java
public static <T extends Comparable<? super T>> Comparator<T> naturalOrder() {
    return (Comparator<T>) Comparators.NaturalOrderComparator.INSTANCE;
}
```

和`reverseOrder`类似，返回一个单例对象，不过这回这个对象不是定义在 Collections 中，而是定义在 Comparators 类内部，Comparators 是 Comparator 的支持类：

```java
// 该枚举类定义于 Comparators 类中
enum NaturalOrderComparator implements Comparator<Comparable<Object>> {
    INSTANCE;

    @Override
    public int compare(Comparable<Object> c1, Comparable<Object> c2) {
        return c1.compareTo(c2);
    }

    @Override
    public Comparator<Comparable<Object>> reversed() {
        return Comparator.reverseOrder();
    }
}
```

通过`return c1.compareTo(c2)`可以看到默认自然排序就是升序。

#### 逆序

上面讨论了，自然排序（natural order）和自然排序的逆序（reverse order），Collections 类和 Comparator 类，当中各有一个`reverse`和`reversed`方法，需要区分开来。

先讨论`Collections.reverse()`方法，从语义上讲 reverse 动词，而它又是静态方法，所以可以推测出它是作用在某个对象上，然后进行修改，就和`Collections.sort()`一样。

```java
public static void reverse(List<?> list) {
    int size = list.size();
    if (size < REVERSE_THRESHOLD || list instanceof RandomAccess) {
        for (int i=0, mid=size>>1, j=size-1; i<mid; i++, j--)
            swap(list, i, j);
    } else {
        // instead of using a raw type here, it's possible to capture
        // the wildcard but it will require a call to a supplementary
        // private method
        ListIterator fwd = list.listIterator();
        ListIterator rev = list.listIterator(size);
        for (int i=0, mid=list.size()>>1; i<mid; i++) {
            Object tmp = fwd.next();
            fwd.set(rev.previous());
            rev.set(tmp);
        }
    }
}
```

通过源码看出，这个被修改的对象，就是一个 List，整个修改的过程就是遍历+交换，达到列表翻转的效果。

而`Comparator.reversed`从语义上看，reversed 是个修饰词，它修饰的，毫无疑问就是一个比较器，reversed comparator，即”翻转的比较器“，或者称”逆序的比较器“。

```java
// 该方法定义在 java.util.Comparator 中
default Comparator<T> reversed() {
    return Collections.reverseOrder(this);
}

// 该方法定义在 java.util.Collections 中
public static <T> Comparator<T> reverseOrder(Comparator<T> cmp) {
    // cmp 为空，返回默认的 ReverseComparator.REVERSE_ORDER
    if (cmp == null) {
        return (Comparator<T>) ReverseComparator.REVERSE_ORDER;
    // 如果 cmp 是 REVERSE_ORDER，就返回自然排序比较器的单例对象
    } else if (cmp == ReverseComparator.REVERSE_ORDER) {
        return (Comparator<T>) Comparators.NaturalOrderComparator.INSTANCE;
    // 如果 cmp 是自然排序比较器的单例对象，就返回 REVERSE_ORDER
    } else if (cmp == Comparators.NaturalOrderComparator.INSTANCE) {
        return (Comparator<T>) ReverseComparator.REVERSE_ORDER;
    // 其他情况
    } else if (cmp instanceof ReverseComparator2) {
        return ((ReverseComparator2<T>) cmp).cmp;
    } else {
        return new ReverseComparator2<>(cmp);
    }
}
```

这里调用了`Collections.reverseOrder()`方法，它是前面提到的`reverseOrder()`的重载版本，之前的没有参数，返回一个实例对象`ReverseComparator.REVERSE_ORDER`，而这里的`reverseOrder()`接受一个`Comparator`对象，如果传入的比较器为空，同样返回`ReverseComparator.REVERSE_ORDER`，剩下的`else if`也不难理解，在注释中做了简单的描述。

---

## 流中的排序

Stream 也是 JDK1.8 的新特性，如果说 Array 和 Collection 是负责存储数据与修改数据的，那么流就是负责描述一连串的流水线式的计算过程。