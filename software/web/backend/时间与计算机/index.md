---
title: "时间与计算机"
date: 2021-11-27T10:58:15+08:00
summary: "计算机与时间的一些概念性问题的讨论"
featured_image: "images/background.jpg"
toc: true
---

## 前言

前几天在工作中碰到与时间有关的 Bug，捣鼓了很久，虽然最后搞好了，但是总觉得对于时间的概念以及对应的 API 接口都有些模糊，特别是一些时区，时间戳，Java8 中的时间工具，都在此篇中作整理，以备不时之需。

---

## 年、月、日

* 一天：地球自转一周的时间

* 一月：月球绕地球转一周的时间，30 天

* 一年：地球绕太阳公转一周的时间，365 天

人们日出而作、日落而息，自古以来，这便是规律，通过观察这些规律，人们就定出了“年月日”、“春夏秋冬”。所以一开始，对于时间的概念就是来源于观察而得的天文现象，和农业密切挂钩，实用但是略显粗糙。

---

## 时、分、秒

这里我并未做太深入的调查，而且我的地理学的很不好，但我自己的想法是，在 15 - 17 世纪，地理大发现的时代，人们对于地理位置、时间精度的要求更高了，就需要对于时间的概念更加精细化。在 1660 年，伦敦皇家学会提出了，在地球表面，摆长约一米的单摆，一次摆动或是半周期（没有反复的一次摆动）的时间大约是一秒。而 60 秒等于一分钟，60 分钟等于一小时，24 个小时就是一天，这都是众所周知的事情了。

总的来说，技术伴随着需求的进步而不断发展，对于时间精度的要求也是跟着人们的生产需求不断提高，再到 19 - 20 世纪，人们就不满足于这样简单的定义一秒钟了，人们发现地球公转的轨道并不是一个正圆形，而是椭圆，速度是不一样的，随着技术进步，人们还观测到地球的自转也不是固定的时间，受到地球潮汐力，地壳运动等等自然现象的影响，就导致了年月日的时间不是固定的。

假设我们以天文现象作为标准，1 年 = 365 天，1 天 = 24 小时，1 小时 = 60 分钟，1 分钟 = 60 秒，1 秒 = 1000 毫秒。结果，现在 1 年不一定是 365 天，一天也不一定是 24 小时，这样换算出来的秒就不对了。这样就会产生时间误差，如果是千百年前的人们，其实生活、劳动、种植、商贸都不会受这点误差的影响，但是现在所有的科技产品（火箭、炮弹、因特网、体育比赛）都需要依赖于非常精确的时间，所以人们就把目光从宏大的天体转向了微观的粒子。

## 原子钟

到了近代，人们的需求从简单的知道大概处于一天中的哪个时间段，一年中的哪个季节，哪个月份，演变成了需要一个精确的“1 秒钟”。怎么定义这个精确的“1 秒钟”呢，天体是不靠谱了，因为上述所说的种种天文气象地理原因，都无法做到定义一段固定的时间。但是，就像上面提到的钟摆，从机械运动的角度来说，就可靠多了，从日晷到摆钟，再到机械钟和石英钟，通过周期性的运动，使得一段时间的定义越来越精确。

目前世界上最准确的计时工具就是原子钟，它是20世纪50年代出现的，原子钟是利用原子吸收或释放能量时发出的电磁波来计时的。由于这种电磁波非常稳定，再加上利用一系列精密的仪器进行控制，原子钟的计时就可以非常准确了。现在用在原子钟里的元素有氢（Hydrogen）、铯（Cesium）、铷（Rubidium）等。原子钟的精度可以达到每2000万年才误差1秒。这为天文、航海、宇宙航行提供了强有力的保障。

---

## 世界标准时间

现在，我们可以看到，已经有了两套计时标准：

1. 原子时：基于原子钟，精确的给出一秒的时间。
2. 世界时：基于天文学 + 时钟计时，和地球自转公转相匹配。

既然，原子钟如此精确，那能否替代掉自古历来通过天文观察的世界时呢？答案是否定的，如果完全采用原子时，随着地球自传不断变慢，可能几百年以后的某一天，正值太阳高高挂在头顶上，原子钟给出的时间已经是下午 4 点了，再过个几千年，原子钟给出的时间就是晚上 8 点了，因为原子钟是不会偏差的，但是天体运动会有偏差，这就和人们自古以来的生活习惯完全不符了。

所以很多科学技术都是以人为本，服务于人类，有点像网络中的 DNS，如果人都能有很好的记忆力，就都用 IP 地址上网了，又或者像计算机编程中，如果人都能用 0 和 1 写代码，也就不需要那些个高级语言、设计模式，但实际是，人不能做到像机器一样，所以采用了很多围绕人产生的各种技术。

时间也是如此，为了兼顾基于天文测量的世界时，人类会持续观测世界时与这个新时钟的差距，也就是建立一条新的标准，不断纠正，原子时和世界时的误差。简单的说就是，如果地球转速变慢或者变快了，就让原子钟加一秒或者减一秒，这个一秒就是“闰秒”，当世界时和原子时差了 0.9 秒，人们就会手动调整原子钟的时间。

往大了说，采用这样的手段，才能延续人类的文明与文化，几千年以后的人们才会理解我们现在有关时间的任何信息。

由于这个时钟是基于原子时 + 世界时「协调」得出的，所以科学家们把它定义为协调世界时（Coordinated Universal Time，简称 UTC）。

---

## GMT 和 UTC

UTC（Coordinated Universal Time） 就是上面说的协调世界时，是以原子时秒长为基础，在时刻上尽量接近于世界时的一种时间计量系统，是一个时间参考标准。而 GMT 是格林威治时间（Greenwich Mean Time）是指位于伦敦郊区的皇家格林尼治天文台的标准时间，因为本初子午线被定义在通过那里的经线，两者的区别在于，UTC 是标准时间，而 GMT 是时区时间，不同地区由于处于不同的经度上，所以各个时区的时间是不一样的。可以用 UTC + 差的小时数来得到当地时区的时间。

GMT 处于本初子午线上，是 0 时区，所以 GMT  = UTC + 0 小时。

CST（China Standard Time）是中国标准时间，由于我国处于东八区，所以 CST = GMT + 8 小时 = UTC + 8 小时。

所以 GMT 和 UTC 从性质而言是不一样的，但是时间的数值是相同的。

---

## 时区

时区是地球上的区域使用同一个时间定义。以前，人们通过观察太阳的位置（时角）决定时间，这就使得不同经度的地方的时间有所不同（地方时）。1863年，首次使用时区的概念。时区通过设立一个区域的标准时间部分地解决了这个问题。

世界各个国家位于地球不同位置上，因此不同国家，特别是东西跨度大的国家日出、日落时间必定有所偏差。这些偏差就是所谓的时差。

为了照顾到各地区的使用方便，又使其他地方的人容易将本地的时间换算到别的地方时间上去。有关国际会议决定将地球表面按经线从东到西，划成一个个区域，并且规定相邻区域的时间相差1小时。在同一区域内的东端和西端的人看到太阳升起的时间最多相差不过1小时。当人们跨过一个区域，就将自己的时钟校正1小时（向西减1小时，向东加1小时），跨过几个区域就加或减几小时。这样使用起来就很方便。

现今全球共分为24个时区。由于实用上常常1个国家，或1个省份同时跨着2个或更多时区，为了照顾到行政上的方便，常将1个国家或1个省份划在一起。所以时区并不严格按南北直线来划分，而是按自然条件来划分。例如，中国幅员宽广，差不多跨5个时区，但为了使用方便简单，实际上在只用东八时区的标准时即北京时间为准。

---

## Unix 时间戳

Unix 时间戳是从1970年1月1日（UTC/GMT的午夜）开始所经过的秒数，不考虑闰秒。系统用的是 32 位有符号整型存储，所以最大是 2147483647 秒，也就是格林尼治时间 2038 年 1 月 19 日凌晨 03:14:07 就会溢出，变成了第 -2147483648 秒（代表格林尼治时间1901年12月13日20:45:52）。

---

## Java 中的时间

### 历史遗留

经常能在项目中看到对时间的操作，用的是`java.util.Date`和`java.util.Calendar`，就如《Java 8 实战》第十二章所述的那样，使用这两个 API 有诸多的缺点和不便之处，我在一开始学习 Java SE 的时候，对时间这一块就很迷惑，本来时间的处理就很复杂了，这两个 API 让时间处理变得更加麻烦。然而令人痛苦的是，不少项目依然使用这两个类处理时间，为以后的维护增加难度。

### New API

Java 8 提供了新的时间处理类，包括`LocalDate`，`LocalTime`，`LocalDateTime`，`Instant`，`Duration`，`Period`等等。与 Date、Calendar 不一样的是，Java 8 中的这些 API，创建出来的日期-时间类，是不可变的，这是为了更好的支持函数式编程、线程安全。

### LocalDate、LocalTime

Java 8 中的时间 API 从面向角度可以分成两部分，一个是面向人的，可读性强，包括`LocalDate`，`LocalTime`，`LocalDateTime`，另一部分是以时间戳为基础的`Instant`，对机器更加友好。

LocalDate 源码的注释信息：

> A date without a time-zone in the ISO-8601 calendar system, such as 2007-12-03.
>
> LocalDate is an immutable date-time object that represents a date, often viewed as year-month-day. Other date fields, such as day-of-year, day-of-week and week-of-year, can also be accessed. For example, the value "2nd October 2007" can be stored in a LocalDate.
>
> This class does not store or represent a time or time-zone. Instead, it is a description of the date, as used for birthdays. It cannot represent an instant on the time-line without additional information such as an offset or time-zone.
>
> The ISO-8601 calendar system is the modern civil calendar system used today in most of the world. It is equivalent to the proleptic Gregorian calendar system, in which today's rules for leap years are applied for all time. For most applications written today, the ISO-8601 rules are entirely suitable. However, any application that makes use of historical dates, and requires them to be accurate will find the ISO-8601 approach unsuitable.
>
> This is a value-based class; use of identity-sensitive operations (including reference equality (==), identity hash code, or synchronization) on instances of LocalDate may have unpredictable results and should be avoided. The equals method should be used for comparisons.
> Implementation Requirements:
> This class is immutable and thread-safe.

翻译：

> ISO-8601 日历系统中没有时区的日期，例如 2007-12-03。 
>
> LocalDate 是表示日期的不可变日期-时间对象，通常被视为年-月-日。也可以访问其他日期字段，例如一年中的某一天、一周中的某一天和一年中的某一周。例如，“2nd October 2007”可以存储在 LocalDate 中。
>
> 此类不存储或表示时间或时区。相反，它是对日期的描述，比如用作表示生日。如果没有偏移量或时区等附加信息，它就无法表示时间线上的瞬间。
>
> ISO-8601 日历系统是当今世界大部分地区使用的现代民用日历系统。它相当于预兆公历系统，其中今天的闰年规则适用于所有时间。对于当今编写的大多数应用程序，ISO-8601 规则完全适用。但是，任何使用历史日期并要求它们准确的应用程序都会发现 ISO-8601 方法不合适。
>
> 这是一个基于值的类；在 LocalDate 的实例上使用身份敏感操作（包括引用相等 (==)、身份哈希码或同步）可能会产生不可预测的结果，应该避免。应该使用 equals 方法进行比较。
>
> 实现要求：这个类是不可变的和线程安全的。

LocalDate 提供了简单的日期，并不包含当天的时间信息，并且它的实例是不可变对象。我们可以通过静态工厂方法`of`来创建一个 LocalDate 实例：

```java
LocalDate localDate1 = LocalDate.of(2021, 11, 21);
LocalDate localDate2 = LocalDate.of(2021, Month.of(11), 21);
```

也可以获取每个字段的值

```java
int year = localDate1.getYear();
Month month = localDate1.getMonth();
int monthValue = localDate1.getMonthValue();
int dayOfMonth = localDate1.getDayOfMonth();
DayOfWeek dayOfWeek = localDate1.getDayOfWeek();
int dayOfYear = localDate1.getDayOfYear();
```

 除了上面这种直观的 getOf 形式，还有一个工厂方法，get(TemporalField field) 可以通过指定字段，来获取值。

```java
int year = localDate1.get(ChronoField.YEAR);
int month = localDate1.get(ChronoField.MONTH_OF_YEAR);
int day = localDate1.get(ChronoField.DAY_OF_MONTH);
```

LocalTime 和 LocalDate 的用法相似，也有 of 的静态工厂方法提供生成实例，以及对应的 get 方法

```java
LocalTime localTime = LocalTime.of(13, 39, 12);

int hour = localTime.getHour();
int minute = localTime.getMinute();
int second = localTime.getSecond();
```

---

### LocalDateTime

LocalDateTime 是 LocalDate 和 LocalTime 的合体，可以同时表示日期和时间，但不带时区信息。

可以直接创建 LocalDateTime，也可以通过合并 LocalDate 和 LocalTime 来创建 LocalDateTime

```java
LocalDateTime localDateTime1 = LocalDateTime.of(2021, 11, 21, 13, 14, 44);
LocalDateTime localDateTime2 = LocalDateTime.of(localDate1, localTime);
```

有了合并自然有拆分，可以从一个 LocalDateTime 实例中，拆成一个 LocalDate 实例和一个 LocalTime 实例

```java
LocalDate localDate = localDateTime1.toLocalDate();
LocalTime localTime = localDateTime1.toLocalTime();
```

---

### Instant

Instant 主要用于处理时间戳，和上面几个类一样的是，它也通过静态工厂方法创建实例，ofEpochSecond 方法有两个重载版本，一个接收秒数，一个接收秒数和纳秒数。

```java
Instant instant = Instant.ofEpochMilli(1000);
Instant instant1 = Instant.ofEpochSecond(1);
Instant instant2 = Instant.ofEpochSecond(1, 0);
```

以上的几个 Instant 实例都表示：1970-01-01T00:00:01Z

---

### Duration 和 Period

当我们需要表示时间间隔的时候，就需要用到这两个类。

* Duration：用来度量秒和纳秒之间的时间值
* Period：用来度量年月日和几天之间的时间值

可以通过 between 方法传入两个时间（实现了 Temporal 接口），上面提到的 LocalDate、LocalTime、LocalDateTime、Instant 都实现了这个接口，所有都可以放入。

```java
LocalDateTime localDateTime1 = LocalDateTime.of(2021, 11, 27, 10, 0);
LocalDateTime localDateTime2 = LocalDateTime.of(2021, 11, 27, 11, 0);
Duration duration = Duration.between(localDateTime1, localDateTime2);
```

除了通过传入两个 temporal 对象，还可以通过 of 创建 Duration 实例：

```java
Duration duration1 = Duration.ofDays(1);
Duration duration2 = Duration.ofHours(1);
Duration duration3 = Duration.ofMinutes(1);
Duration duration4 = Duration.ofMillis(1);
Duration duration5 = Duration.ofSeconds(1);
```

---

### 改变时间

上述创建的时间对象，都是不可被修改的，但是可以通过 withAttribute 方法来创建对象的副本，在副本中修改相应的属性

```java
LocalDate localDate1 = LocalDate.of(2021, 11, 27); // 2021-11-27
LocalDate localDate2 = localDate1.withYear(2020); // 2020-11-27
LocalDate localDate3 = localDate1.withMonth(10); // 2021-10-27
```

除了直接指定对应属性的值，还可以通过声明式的增加和减少来操纵时间

```java
LocalDate localDate4 = localDate1.plusMonths(3); // 2022-02-27
LocalDate localDate5 = localDate1.minusYears(2); // 2019-11-27
```

---

### 格式化时间

通过我们需要把日期-时间打印出来或者转成不同格式的字符串，比如 2021 年 11 月 27 日可以表达成以下不同格式的字符串：

* 2021-11-27
* 20211127
* 2021_11_27
* 2021/11/27
* 11/27 2021

通过 DateTimeFormatter 可以很方便的实现格式转换的功能：

```java
LocalDate localDate = LocalDate.of(2021, 11, 27);
String date1 = localDate.format(DateTimeFormatter.ofPattern("yyyy-MM-dd"));
String date2 = localDate.format(DateTimeFormatter.ofPattern("yyyyMMdd"));
String date3 = localDate.format(DateTimeFormatter.ofPattern("yyyy_MM_dd"));
String date4 = localDate.format(DateTimeFormatter.ofPattern("yyyy/MM/dd"));
String date5 = localDate.format(DateTimeFormatter.ofPattern("MM/dd yyyy"));
```

在 DateTimeFormatter 中，内置了一些常量，比如 BASIC_ISO_DATE，ISO_lOCAL_DATE，ISO_LOCAL_DATE_TIME，ISO_TIME 等等，不过我更倾向于 ofPattern，能够直接看到格式。

---

### 时间字符串的解析

将日期-时间对象格式化成字符串， 同样的也可以把字符串解析成对象。

```java
DateTimeFormatter formatter = DateTimeFormatter.ofPattern("yyyy-MM-dd");
LocalDate localDate = LocalDate.of(2021, 11, 27);
String date1 = localDate.format(DateTimeFormatter.ofPattern("yyyy-MM-dd"));
LocalDate localDate1 = LocalDate.parse(date1, formatter);
```

