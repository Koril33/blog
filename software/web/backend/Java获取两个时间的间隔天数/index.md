---
title: "Java获取两个时间的间隔天数"
date: 2023-12-23T11:37:26+08:00
tags: []
featured_image: "images/background.jpg"
summary: "Period 获取间隔天数的错误用法"
toc: true
---

## 前置知识

Java8 中使用 `java.time.Duration` 和 `java.time.Period` 来对时间间隔（时间段）进行抽象，其中 Duration 类的基本单位是秒和纳秒，而 Period 则面向更长的时间段，基本单位是年、月、日。

另外，Java8 Time API 中还有一个类——`java.time.temporal.ChronoUnit`用于对时间单位建模，它是一个枚举类，包含了所有常见的基本时间单位。

---

## 一个错误的用法

接下来就是一个简单的需求，计算两个 LocalDateTime 之间的间隔天数：

```java
LocalDateTime t1 = LocalDateTime.of(LocalDate.of(2023, 12, 12), LocalTime.MIN);
LocalDateTime t2 = LocalDateTime.of(LocalDate.of(2023, 12, 13), LocalTime.MIN);

// 如何计算出 t1 和 t2 之间的时间间隔？
```

然后，我似乎理所当然的调用了 Period 类的 getDays() 方法：

```java
// Period 的基本单位是天，所以 LocalDateTime 需要转成 LocalDate
Period period = Period.between(t1.toLocalDate(), t2.toLocalDate());
System.out.println(period.getDays());
```

返回的结果是 1，2023 年 12 月 12 日和 2023 年 12 月 13 日确实相差一天，看起来似乎没问题。

但这里建立的前提是，我以为 getDays() 返回的是两日期之间间隔的秒数 / 86400 后，类似下面的代码：

```java
long s1 = t1.atZone(ZoneId.systemDefault()).toInstant().getEpochSecond();
long s2 = t2.atZone(ZoneId.systemDefault()).toInstant().getEpochSecond();
System.out.println((s2 - s1) / 86400);
```

但是只要稍微改一下日期中的年份或者月份就不对了，比如，将 t1 中的月份改成 11 月，理论上，此时 t1 和 t2 的间隔天数应该是 31 天：

```java 
LocalDateTime t1 = LocalDateTime.of(LocalDate.of(2023, 11, 12), LocalTime.MIN);
LocalDateTime t2 = LocalDateTime.of(LocalDate.of(2023, 12, 13), LocalTime.MIN);

Period period = Period.between(t1.toLocalDate(), t2.toLocalDate());
// Period 算出来的结果居然是 1
System.out.println(period.getDays());

long s1 = t1.atZone(ZoneId.systemDefault()).toInstant().getEpochSecond();
long s2 = t2.atZone(ZoneId.systemDefault()).toInstant().getEpochSecond();
// 使用秒数计算，结果正常，是 31 天
System.out.println((s2 - s1) / 86400);
```

问题浮现出来了，Period 的建模是针对间隔时间的年，月，日，它的 getDays 并不是计算两个日期之间的总天数差，而是取出 Period 对象的 days 这个字段，我们把另外两个参数（年和月）也打印出来就能看到原因了：

```java
LocalDateTime t1 = LocalDateTime.of(LocalDate.of(2023, 11, 12), LocalTime.MIN);
LocalDateTime t2 = LocalDateTime.of(LocalDate.of(2023, 12, 13), LocalTime.MIN);

Period period = Period.between(t1.toLocalDate(), t2.toLocalDate());
System.out.println("period: " + period);
System.out.println("years: " + period.getYears());
System.out.println("months: " + period.getMonths());
System.out.println("days: " + period.getDays());
```

打印结果如下：

```
period: P1M1D
years: 0
months: 1
days: 1
```

Period 的 months 和 days 都是 1，表示 2023 年 11 月 12 日和 2023 年 12 月 13 日，相差一个月多一天。

所以不能用 getDays() 获取实际间隔天数。

## 一些正确用法

第一种，就是上面提到的，将 LocalDate 转换成对应的时间戳，相减，然后除以 86400 得到天数。

第二种，是用 Period 获得的年月日参数，来计算天数，但这种方法很麻烦，因为需要考虑闰年和每个月的具体天数，非常麻烦，所以基本上不会使用。

第三种，使用 LocalDate 的 toEpochDay：

```java
LocalDateTime t1 = LocalDateTime.of(LocalDate.of(2023, 11, 12), LocalTime.MIN);
LocalDateTime t2 = LocalDateTime.of(LocalDate.of(2023, 12, 13), LocalTime.MIN);
// 转换成 epoch day, 即相对于 1970 年 1 月 1 日的天数
long epochDay1 = t1.toLocalDate().toEpochDay();
long epochDay2 = t2.toLocalDate().toEpochDay();

System.out.println("epochDay1: " + epochDay1 + ", epochDay2: " + epochDay2);
System.out.println("Between days: " + (epochDay2 - epochDay1));
```

打印结果：

```
epochDay1: 19673, epochDay2: 19704
Between days: 31
```

第四种，使用 LocalDateTime 的 until() 方法：

```java
LocalDateTime t1 = LocalDateTime.of(LocalDate.of(2023, 11, 12), LocalTime.MIN);
LocalDateTime t2 = LocalDateTime.of(LocalDate.of(2023, 12, 13), LocalTime.MIN);

System.out.println("Between Days: " + t1.until(t2, ChronoUnit.DAYS));
```

打印结果：

```
Between Days: 31
```

第五种，使用 ChronoUnit 枚举类的 between() 方法：

```java
LocalDateTime t1 = LocalDateTime.of(LocalDate.of(2023, 11, 12), LocalTime.MIN);
LocalDateTime t2 = LocalDateTime.of(LocalDate.of(2023, 12, 13), LocalTime.MIN);

System.out.println("Between Days: " + ChronoUnit.DAYS.between(t1, t2));
```

打印结果：

```
Between Days: 31
```

---

## 结论

从简洁性来看，最推荐的是第四种和第五种方法，在业务代码中编写涉及到时间 API 的代码时，需要反复确认和测试，不然很容易引发意料之外的 BUG。

