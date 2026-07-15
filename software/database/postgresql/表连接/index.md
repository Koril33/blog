---
title: "表连接"
date: 2026-06-06T10:09:03
summary: "表连接的概念，类型，使用时机"
---

## 目录

[TOC]

---

## 前言

刚毕业做 CRUD boy 的时候，曾听前辈说最好不要使用关系型数据库的表连接，也就是多表的 JOIN，这个问题困扰了我很长时间，因为大部分时间都花在了编写实现公司业务逻辑的代码上，所以也就随波逐流，没有深究。

现在回看这个问题，一刀切是有失偏颇的，使用还是不使用 JOIN，都要看具体的业务和使用场景。

---

## 表连接的误解

因为不了解底层的机制和一些坑（缺少专业的 DBA），在很多项目组里，我们会把关系型数据库用成了非关系型数据库，不敢用外键约束，不敢使用连表查询，不敢使用触发器和视图，可能仅仅在单表数据量超过百万的时候，发现接口慢了，才去加个索引。

有几个原因导致了这种情况：

1. 团队没有了解数据库的 DBA，无法在业务建模时提供专业的建议。
2. 需求分散且多变，互联网产品尤其如此，朝令夕改的模式并不适合稳定的关系型数据库的建模。
3. 多用多错，很多业务下确实只用到了单表查询这种简单的功能，如果用了复杂的功能，出了问题很难解决。
4. 遗留的历史包袱，很多项目从十年前（甚至更久）立项，经过了几代程序员、产品经理、项目经理以及领导的迭代，持之以恒的堆砌屎山，缺乏合理的数据库建模和架构。

---

## 表连接的作用

表连接是关系型数据库的灵魂之一，它解决了数据的组织，维护和一致性的问题。

我们以一个最常见的用户-订单的业务举例子，用户购买了一个商品，产生了一个订单，假设不使用关系模型，把所有信息塞在一个表里：

```sql
create table user_product_order
(
    username      text,
    phone         text,
    address       text,
    product       text,
    number        integer,
    product_price integer,
    total_price   integer,
    created_at    timestamptz
);

comment on column user_product_order.username is '用户名';
comment on column user_product_order.phone is '用户手机号码';
comment on column user_product_order.address is '用户地址';
comment on column user_product_order.product is '商品名';
comment on column user_product_order.number is '购买数量';
comment on column user_product_order.product_price is '单价';
comment on column user_product_order.total_price is '总价';
comment on column user_product_order.created_at is '订单创建时间';
```

示例数据：

```
```

这种单表存储所有数据（包括了两个实体类的信息数据），有一些问题：

1. 实体类没有主键字段，无法唯一的在数据库系统标志一个用户，主键是唯一且不能改变的。
2. 如果实体类更改了信息，比如用户 alice 修改了手机号，商品价格发生变动，我们需要扫描整张订单表去修改相关的数据，因为所有的实体类信息都被冗余存储了，alice 下了 1000 次订单，那么她的个人信息就在这张表里出现了 1000 次，她修改了手机号就需要把 1000 条订单记录都修改一遍。
3. 会出现删除异常的问题，假如用户 alice 的所有订单都被删除了，那么她作为用户本身也消失了。
4.
