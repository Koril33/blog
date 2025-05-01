---
title: "JPA中的枚举类型字段存储"
date: 2023-08-20T14:49:30+08:00
tags: []
featured_image: "images/background.jpg"
summary: "JPA + Java enum 存储枚举类型的字段"
toc: true
---

## 前言

在 JPA 版本 2.0 及更低版本中，没有将枚举值映射到数据库字段的便捷方法。每个选项都有其局限性和缺点，但这些问题都可以通过使用 JPA 2.1 功能部件来避免。

在本文中，将介绍使用 JPA 在数据库中持久化枚举的不同可能性。还将描述它们的优缺点，并提供简单的代码示例。

---

## 注解 @Enumerated

在 JPA 2.1 之前，将枚举类型（enum）字段值映射到数据库中，最常见的做法是使用 @Enumerated 注解，通过这种方式，可以让 JPA 将枚举值转换成字符串形式或者它的序数（ordinal）。

下面演示这两种不同的方式。首先创建一个 Article 类和 article 表方便演示：

```java
@Data
@Entity
@Table(name = "article")
public class Article {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    @Column(name = "id")
    private Integer id;


    @Column(name = "title")
    private String title;
}
```

```sql
create table `demo-jpa-enum`.article
(
    id    int auto_increment
        primary key,
    title varchar(50) default '0' not null
);
```

### 方式一：映射成序数

如果我们将`@Enumerated(EnumType.ORDINAL)`注释放在枚举字段上，JPA 在数据库中持久化给定实体时将使用`Enum.ordinal()`的值。

假设文章有以下枚举状态：

```java
public enum Status {
    OPEN, REVIEW, APPROVED, REJECTED;
}
```

将 status 字段添加到 Article 中：

```java
@Data
@Entity
@Table(name = "article")
public class Article {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    @Column(name = "id")
    private Integer id;


    @Column(name = "title")
    private String title;
    
    
    @Enumerated(EnumType.ORDINAL)
    @Column(name = "status")
    private Status status;
}
```

然后保存一个 article 对象到数据库中：

```java
Article article = new Article();
article.setTitle("first article");
article.setStatus(Status.OPEN);
Article save = articleRepository.save(article);
```

存储会触发以下的 SQL 命令：

```sql
insert 
into
    article
    (status, title) 
values
    (?, ?)
binding parameter [1] as [INTEGER] - [0]
binding parameter [2] as [VARCHAR] - [first article]

```

可以看到数据库存储的是 int 类型的 0，也就是 Status.OPEN.ordinal() 的值，这种方式有个很大的缺点：如果在枚举类的类型中添加新的类型，那么存储到数据库中的 ordinal 值的含义都将发生改变，这意味着我们需要更新整个数据库的 status 字段的值。

打个简单的比方，现在 OPEN 代表 0，如果之后在 Status 类中再添加一个枚举值——CLOSE，并且置于 OPEN 之前，那么数据库中的所有的 status 为 0 的记录，它们的含义都将从 OPEN 变成 CLOSE，如果代码中有用到这个 status 字段做逻辑判断，就会导致一些不可预知的 BUG 产生。

所以使用 ordinal 的值存储到数据库中，并不是一种好的办法。

### 方式二：映射成 String

枚举类型除了 ordinal 之外还有个 name 方法，所以当我们使用`@Enumerated(EnumType.STRING)`来注释枚举类型的时候，JPA 会自动使用`Enum.name()`。

创建第二个枚举类，并添加到 Article 中：

```java
public enum Type {
    INTERNAL, EXTERNAL;
}

@Data
@Entity
@Table(name = "article")
public class Article {

    // 其他字段
    // ...

    @Enumerated(EnumType.STRING)
    @Column(name = "type")
    private Type type;
}
```

存储新的文章到数据库：

```java
Article article = new Article();
article.setTitle("second article");
article.setStatus(Status.APPROVED);
article.setType(Type.INTERNAL);
Article save = articleRepository.save(article);
```

将触发以下 SQL：

```sql
insert 
    into
        article
        (status, title, type) 
    values
        (?, ?, ?)
binding parameter [1] as [INTEGER] - [2]
binding parameter [2] as [VARCHAR] - [second article]
binding parameter [3] as [VARCHAR] - [INTERNAL]
```

可以看到，数据库中该记录的 type 字段存储了字符串——INTERNAL。

存储字符串的方式，解决了前一个使用 ordinal 的方式的缺点，添加新的枚举类型，打乱它们的顺序，数据库存储记录的含义都不会改变，但是如果重命名就不行了，如果 INTERNAL 被重命名为其他名称，我们不得不再将数据库中原来 type 值为 INTERNAL 的记录更新为新的名称。

这种方式的可读性更强，但性能会比使用 ordinal 的方式差一些。

---

## 使用 @PrePersist 和 @PostLoad 注解

另一种方式，是使用 JPA 的回调方法，首先在实体类中添加两个字段来表示一个枚举类型字段，一个是映射到数据库的值字段，一个是标注`@Transient`注解的持有真实枚举类型的字段，然后通过 JPA 回调方法来填充这两个字段的值。

这里新建一个枚举类来演示这个方法：

```java
public enum Priority {

    LOW(100),
    MEDIUM(200),
    HIGH(300);

    Priority(int priority) {
        this.priority = priority;
    }

    private final Integer priority;

    public int getPriority() {
        return priority;
    }

    // of 方法是为了方便能够通过 priority 值获取到对应的 Priority 枚举对象
    public static Priority of(Integer priority) {
        return Arrays.stream(Priority.values())
                     .filter(
                         p -> Objects.equals(p.getPriority(), priority)
                     )
                     .findFirst().orElseThrow(IllegalArgumentException::new);
    }
}
```

然后添加 priority 字段到 Article 中：

```java
@Data
@Entity
@Table(name = "article")
public class Article {

    // 其他字段
    // ...

    // priorityValue 字段用于与数据库的相互映射，数据库中存储的是 int 类型
    @Basic
    @Column(name = "priority")
    private Integer priorityValue;


    // 不持久化 priority 字段到数据库中，仅仅用于业务代码使用
    @Transient
    private Priority priority;
}
```

priorityValue 用于数据库映射存储，priority 字段用于业务代码逻辑判断使用，两个字段合在一起，表示 priority 的枚举属性。

除此之外，我们还需要加上 JPA 的回调函数：

```java
@Data
@Entity
@Table(name = "article")
public class Article {

    // 其他字段
    // ...

    // priorityValue 字段用于与数据库的相互映射，数据库中存储的是 int 类型
    @Basic
    @Column(name = "priority")
    private Integer priorityValue;


    // 不持久化 priority 字段到数据库中，仅仅用于业务代码使用
    @Transient
    private Priority priority;
    
    
    // 从数据库中加载数据前调用该方法
    @PostLoad
    void fillTransientPriority() {
        if (Objects.nonNull(priorityValue)) {
            this.priority = Priority.of(priorityValue);
        }
    }

    // 持久化到数据库前调用该方法
    @PrePersist
    void fillPersistentPriority() {
        if (Objects.nonNull(priority)) {
            this.priorityValue = priority.getPriority();
        }
    }
}
```

`fillTransientPriority()`确保了在数据库加载 Article 到应用程序中的业务代码的时候，将数据库中存储的 int 类型转成业务代码需要的 Priority 类型。

`fillPersistentPriority`确保了持久化到数据库中存储前，将业务代码中的 Priority 类型的值，转换成 int 类型。

```java
Article article = new Article();
article.setTitle("third article");
article.setStatus(Status.APPROVED);
article.setType(Type.EXTERNAL);
// 这里使用的是 Priority 类型
article.setPriority(Priority.MEDIUM);
Article save = articleRepository.save(article);
```

触发的 SQL：

```sql
insert 
into
    article
    (priority, status, title, type) 
values
    (?, ?, ?, ?)
binding parameter [1] as [INTEGER] - [200]
binding parameter [2] as [INTEGER] - [2]
binding parameter [3] as [VARCHAR] - [third article]
binding parameter [4] as [VARCHAR] - [EXTERNAL]
```

可以看到业务代码中的 Priority.MEDIUM 转换成了它的 priority value——200，这样不仅解决了 Enum.ordinal 的问题，可以随意插入新的枚举值，更换枚举值之间的顺序，而不影响到数据库存储数值对应的枚举类型，并且还解决了 Enum.name 的问题，更换枚举类型的名字（只要不更新 priority value）也没有关系。

缺点就是，通过两个字段来表示一个枚举类型字段，会有些复杂，而且无法在 JPQL 中使用枚举类型。

---

## 使用 JPA 2.1 的 @Converter 注解

为了克服上述解决方案的局限性，JPA 2.1 发行版引入了一个新的标准化 API，可用于将实体属性转换为数据库值，反之亦然。我们需要做的就是创建一个实现`AttributeConverter`的新类，并用 @Converter 对实现类进行注释。

创建一个新的枚举类：

```java
public enum Category {

    SPORT("S"),
    MUSIC("M"),
    TECHNOLOGY("T");

    private final String code;

    Category(String code) {
        this.code = code;
    }

    public String getCode() {
        return code;
    }
}
```

在 Article 中添加 category 字段：

```java
@Data
@Entity
@Table(name = "article")
public class Article {

    // 其他字段
    // ...
    
    private Category category;
}
```

然后编写一个 AttributeConverter 实现类：

```java
import javax.persistence.AttributeConverter;
import javax.persistence.Converter;
import java.util.Arrays;
import java.util.Objects;

// 使用这个注解，在用到 Category 的地方自动转换
// 不加这个注解，需要在实体类相关字段上加上 @Convert
@Converter(autoApply = true)
public class CategoryConverter implements AttributeConverter<Category, String> {
    
    // Java 类转换成数据库类型
    @Override
    public String convertToDatabaseColumn(Category attribute) {
        if (Objects.isNull(attribute)) {
            return null;
        }
        return attribute.getCode();
    }

    // 数据库类型转换成 Java 类型
    @Override
    public Category convertToEntityAttribute(String dbData) {
        if (Objects.isNull(dbData)) {
            return null;
        }
        return Arrays.stream(Category.values())
                .filter(c -> Objects.equals(c.getCode(), dbData))
                .findFirst()
                .orElseThrow(IllegalArgumentException::new);
    }
}
```

AttributeConverter<X, Y> 的 X 表示需要转换的类的类型，Y 表示数据库字段类型。

插入新的 Article：

```java
Article article = new Article();
article.setTitle("fourth article");
article.setStatus(Status.APPROVED);
article.setType(Type.EXTERNAL);
article.setPriority(Priority.MEDIUM);
article.setCategory(Category.MUSIC);
Article save = articleRepository.save(article);
```

触发的 SQL：

```sql
insert 
into
	article
	(category, priority, status, title, type) 
values
	(?, ?, ?, ?, ?)
binding parameter [1] as [VARCHAR] - [M]
binding parameter [2] as [INTEGER] - [200]
binding parameter [3] as [INTEGER] - [2]
binding parameter [4] as [VARCHAR] - [fourth article]
binding parameter [5] as [VARCHAR] - [EXTERNAL]
```

可以看到 Category.MUSIC 被转换成对应的 code——M，存储到了数据库中。

如果不在 CategoryConverter 类上添加 @Converter(autoApply = true)，则需要在实体类上的相关字段手动加上 @Convert：

```java
@Column(name = "category")
@Convert(converter = CategoryConverter.class)
private Category category;
```

该方法，解决了以上所有其他方式的缺点，将数据库中已经存储的数据和 Java 应用中的枚举类的属性给分离开，通过 AttributeConverter 可以安全的添加新的枚举类型以及更改枚举类的名称，没有额外的 @Transient 字段和 JPA 回调函数，将代码封装在了一个 Converter 类中，提供了更多的转换的灵活性。
