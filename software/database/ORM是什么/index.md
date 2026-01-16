---
title: "ORM是什么"
date: 2025-12-12T10:54:27
summary: ""
---

## ORM 的概念

ORM（Object-Relational-Mapping）作为一种软件层面的抽象，提供了一座连接 OOP 语言（例如：Java、C#、Python）和关系型数据库（例如：MySQL、PostgreSQL、SQLite）的桥梁。

在面向对象编程中，对于业务的抽象和建模是依赖于语言提供的对象机制，一个对象包含了一些字段和方法，对象和对象之间的关联关系通过引用或者指针来进行链接。对于数据集的操作（增删改查）依赖于特定的数据结构（例如：set、map、list），这些数据结构的构建、销毁以及查询都在内存级别进行。

而关系型数据库的语言主要是 SQL，关系型数据库提供了关系表的概念来对现实世界的业务问题进行建模，一张表包含了多个列，这些列名可以近似于对象里的属性字段，而表中的一行则表示一个独一无二的记录。

不难发现，面向对象和关系型数据库之间存在很多相似的地方：

1. 类和表
2. 类的属性和表的列
3. 类的实例对象和表的行
4. 对象之间的关联和关系表之间的关联

在较大规模下的 Web 开发中，使用面向对象语言直接编写数据库查询语句比较难以维护，需要大量的 SQL 字符串拼接、转义，不仅容易被 SQL 注入攻击，在应用切换底层数据库时也需要改动大量的 SQL 语句，而且后期对于阅读代码，理解业务也造成比较大的困扰。

在这种情况下，ORM 诞生了，它连接了 Object 和 Relational Table，开发者可以以面向对象语言提供的工具直接对 ORM 抽象后的数据库数据进行 CRUD。

以一个 Student 和 Address 为例，DB 中存在两张表：

```sql

create table t_student
(
    id     serial constraint t_student_pk primary key,
    name   varchar,
    gender char
);

create table t_address
(
    id           serial constraint t_address_pk primary key,
    province     varchar,
    city         varchar,
    street       varchar,
    house_number varchar,
    student_id   integer

	constraint t_address_t_student_id_fk
		references t_student
		on delete cascade
);

```

这是一个简单的一对多示例，一个学生拥有多个地址，在地址表中有一个外键用来表示这一行的地址信息属于哪一个学生的。

考虑以下业务场景：学生进入一个 Web 页面，需要填写表单信息，包含了学生姓名，性别和地址信息。

如果使用 SQL 的方式，则添加学生信息的代码如下：

```java
@Transactional
@PostMapping("/add")
public Integer addStudent(@RequestBody Student student) {
    String insertStudentSql = """
        insert into t_student (name, gender)
        values (:name, :gender)
        returning id
        """;

    Integer studentId = namedParameterJdbcTemplate.queryForObject(
            insertStudentSql,
            Map.of("name", student.getName(), "gender", student.getGender()),
            Integer.class
    );

    String insertAddressSql = """
        insert into t_address (
          province, city, street, house_number, student_id
        )
        values (
          :province, :city, :street, :houseNumber, :studentId
        )
        """;

    for (Address address :  student.getAddresses()) {
        namedParameterJdbcTemplate.update(
                insertAddressSql,
                Map.of(
                        "province", address.getProvince(),
                        "city", address.getCity(),
                        "street", address.getStreet(),
                        "houseNumber", address.getHouseNumber(),
                        "studentId", studentId
                )
        );
    }
    return studentId;
}
```

查看所有学生信息的代码：

```java
@GetMapping("/list")
public List<Student> getAllStudents() {

    String sql = """
        select s.id as student_id,
               s.name,
               s.gender,
               a.province,
               a.city,
               a.street,
               a.house_number
        from t_student as s left join t_address as a on s.id = a.student_id
    """;

    return jdbcTemplate.query(sql, rs -> {

        Map<Integer, Student> studentMap = new LinkedHashMap<>();

        while (rs.next()) {
            int studentId = rs.getInt("student_id");

            Student student = studentMap.computeIfAbsent(studentId, id -> {
                Student s = new Student();
                s.setId(id);
                try {
                    s.setName(rs.getString("name"));
                    s.setGender(rs.getString("gender"));
                } catch (SQLException e) {
                    throw new RuntimeException(e);
                }
                return s;
            });


            // LEFT JOIN 可能没有 address
            if (rs.getString("province") != null) {
                Address addr = new Address();
                addr.setProvince(rs.getString("province"));
                addr.setCity(rs.getString("city"));
                addr.setStreet(rs.getString("street"));
                addr.setHouseNumber(rs.getString("house_number"));
                student.getAddresses().add(addr);
            }
        }

        return new ArrayList<>(studentMap.values());
    });
}

```


这样的业务代码一旦出现在较大规模的项目中，会显得非常难看，并且 Java 这种面向对象的语言（或者别的面向对象语言）的作用变成了过程式的代码，完全服务于 HTTP 参数的接收，拼接 SQL 以及处理 DB API 返回的 Result 结果集。

市面上大部分的 ORM 框架，致力于将底层数据库的复杂性（编写 SQL，处理事务，处理对象-关系映射）封装在框架之中，开发者只需要用 OOP 的代码就可以直接操作数据库的数据集。

ORM 式的代码，类似如下：

添加一个学生信息：

```java

Student s = new Student();
s.setName("Tom");
s.setGender("M");

Address a1 = new Address();
a1.setCity("Shanghai");

Address a2 = new Address();
a2.setCity("Beijing");

s.addAddress(a1);
s.addAddress(a2);

entityManager.persist(s);

```

---

## ORM 的问题

尽管从很多角度来看，对象和关系表存在相似之处，但是并不相同，这里面的范式不匹配（或者叫阻抗）会在使用 ORM 框架中衍生出非常多的问题。

对象的关系，在 OOP 中通过对象的互相引用形成一个图，但是在关系型数据库中，是通过外键链接来表示两个表具有关联关系，在 OOP 中导航到另一个对象很简单，一般是用点符号就可以获取到关联的对象，而关系型数据库则需要使用 Join 语句来处理。

数据类型差异，OOP 的数据类型和关系型数据库支持的数据类型并不匹配，而且把约束映射到 OOP 也是一个难点。

唯一性，如何在 ORM 中表示对象的唯一性？

性能问题，对于复杂的查询，使用 ORM 可能会带来不必要的复杂度和性能下降。

---

## 参考

1. https://en.wikipedia.org/wiki/Object%E2%80%93relational_mapping
2. https://en.wikipedia.org/wiki/Object%E2%80%93relational_impedance_mismatch
3. https://aws.amazon.com/what-is/object-relational-mapping/