---
title: "JPA+MySQL的主键选择"
date: 2023-06-13T17:19:42+08:00
tags: []
featured_image: "images/background.jpg"
summary: "不同的场景，需要选择不同的主键生成策略"
toc: true
---

## 前言

不同场景下，表的主键选择也不同，本文围绕 JPA 和 MySQL 的主键选择做一些介绍。

---

## 主键

主键在数据库中最重要的作用是标志着该行数据在整张表中的唯一性，这样数据库才能定位到某一行数据，所以主键具有唯一（unique）和非空（not null）的特点。

除此之外，某一张表的主键也会作为另外一张表的外键，作表之间的关联用。

主键在应用运行过程中，还具有不可修改的特点，因为一旦修改了某行记录的主键，很可能会导致其他表的数据无法关联，因为很多数据库的设计并没有做到强约束。

---

## 业务还是逻辑作主键？

有时候，一些具有唯一性的业务字段（比如，身份证，订单号）会被用作主键，但这并不是一个最佳的选择，主键必须是逻辑性的而非业务性的，它唯一的作用就是标志某行数据，主键不能具有任何业务性的含义。

业务作为主键有以下的缺点：

1. 当业务发生变化时，有时需要变更主键
2. 涉及多列主键时比较难操作
3. 业务主键往往比较长，所占空间更大，导致更大的磁盘 IO，设计一个兼具易用和性能的业务主键生成方案比较难。
4. 有时候应用在持久化一条记录前，并不能得到业务主键的值，碰到这种场景会非常麻烦。

---

## 自增还是 UUID？

我们常用的 MySQL 引擎时 InnoDB，InnoDB 使用聚簇索引， 数据记录本身被存于主索引（一颗 B+Tree）的叶子节点上。

这就要求同一个叶子节点内（大小为一个内存页或磁盘页）的各条数据记录按主键顺序存放，因此每当有一条新的记录插入时，MySQL 会根据其主键将其插入适当的节点和位置，如果页面达到装载因子（InnoDB 默认为 15/16），则开辟一个新的页（节点）。
如果表使用自增主键，那么每次插入新的记录，记录就会顺序添加到当前索引节点的后续位置，当一页写满，就会自动开辟一个新的页。这样就会形成一个紧凑的索引结构，近似顺序填满。

由于每次插入时也不需要移动已有数据，因此效率很高，也不会增加很多开销在维护索引上。

但采用 UUID 的话，由于每次插入主键的值近似于随机，因此每次新记录都要被插到现有索引页的中间某个位置，MySQL 不得不为了将新记录插到合适位置而移动数据，这样就造成了一定的开销。Mysql 为维护索引可能需要频繁的刷新缓冲，增加了方法磁盘 IO 的次数，而且时常需要对索引结构进行重组织。

自增虽然性能好，但也有缺点：

1. 容易泄露信息，因为是自增的，可以很容易猜到下一条数据的主键值。
2. 分库分表存在问题。

UUID 虽然占用空间大，性能差，但是由于它的随机性，对信息的安全更有优势，并且在分布式环境中可以更好的使用（碰撞的概率非常小）。

---

## JPA的主键策略

在实体类中，使用 @Id 来标志一个字段作为主键，另外生成规则由 @GeneratedValue 来设定。

JPA 提供了四种不同的主键生成策略：TABLE，SEQUENCE，IDENTITY，AUTO。

这四种生成策略定义在了`javax.persistence.GenerationType`中：

```java
package javax.persistence;

/** 
 * Defines the types of primary key generation strategies. 
 * 定义了主键生成的策略类型
 * @see GeneratedValue
 *
 * @since 1.0
 */
public enum GenerationType { 

    /**
     * Indicates that the persistence provider must assign 
     * primary keys for the entity using an underlying 
     * database table to ensure uniqueness.
     * 表明持久化程序必须使用基础数据库的表来为实体分配主键以确保唯一性。
     */
    TABLE, 

    /**
     * Indicates that the persistence provider must assign 
     * primary keys for the entity using a database sequence.
     * 表明持久化程序必须使用数据库序列来为实体分配主键。
     */
    SEQUENCE, 

    /**
     * Indicates that the persistence provider must assign 
     * primary keys for the entity using a database identity column.
     * 表明持久化程序必须使用数据库标识列为实体分配主键。
     */
    IDENTITY, 

    /**
     * Indicates that the persistence provider should pick an 
     * appropriate strategy for the particular database. The 
     * <code>AUTO</code> generation strategy may expect a database 
     * resource to exist, or it may attempt to create one. A vendor 
     * may provide documentation on how to create such resources 
     * in the event that it does not support schema generation 
     * or cannot create the schema resource at runtime.
     * 表明持久化程序应该为特定数据库选择适当的策略。 
     * AUTO 生成策略可能期望数据库资源存在，或者它可能尝试创建一个。
     * 如果供应商不支持模式生成或无法在运行时创建模式资源，
     * 则供应商可能会提供有关如何创建此类资源的文档。
     */
    AUTO
}

```

常用数据库支持生成规则如下：

| 数据库   |          支持的策略          |
| -------- | :--------------------------: |
| Postgres | TABLE AUTO IDENTITY SEQUENCE |
| Oracle   |     TABLE AUTO SEQUENCE      |
| Mysql    |     TABLE AUTO IDENTITY      |

MySQL 中可以在创建表时声明 “AUTO_INCREMENT”，数据库会在新行插入时自动给 ID 赋值，这也叫做 ID 自增长列，所以 MySQL 可以选择 IDENTITY 作为主键生成策略。

Oracle 不支持 ID 自增长列而是使用序列的机制生成主键 ID。对此，可以选用序列（Sequence）作为主键生成策略。

---

## 参考

1. https://www.jianshu.com/p/548cb8cce0dc
2. https://www.jianshu.com/p/ee87671a492b
