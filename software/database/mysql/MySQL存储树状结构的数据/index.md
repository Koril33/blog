---
title: "MySQL存储树状结构的数据"
date: 2023-04-04T19:51:05+08:00
tags: []
featured_image: "images/background.jpg"
summary: "数据库存储树状结构的数据，是后端经常碰见的功能需求"
toc: true
---

## 前言

在后端的开发中，经常碰到一些树状结构的模型，比如：公司的部门结构组织、国家的行政区划、产品目录，等等。

当我们需要将这些数据持久化到关系型数据库中，就没有像通用性编程语言一样，可以利用指针或者引用的方式得到一种可以递归的树状结构。

本文简单讨论树状结构在 MySQL 这种关系型数据库的存储方式，尽量保持简洁，通用。

---

## 框架和数据库选择

1. SpringBoot
2. SpringMVC
3. Spring Data JPA
4. MySQL

---

## 项目用例选择

在本文中，选择中国的行政区划作为案例，因为行政区划依照省-市-区划分，是一棵完整的分层的树状结构。

中国行政区划的数据源，参考国家标准：《[GB/T 2260-2007 中华人民共和国行政区划代码](https://www.biaozhun.org/guojia/22345.html)》

---

## Entity

本文的 Entity 类非常简单，仅仅包含构造树结构必要的字段，在实际业务场景中，可以随意增加业务字段。

id 的生成，考虑到通用性，我使用 UUID 作为主键生成策略。

Area.java

```java
import lombok.Data;
import org.hibernate.annotations.GenericGenerator;

import javax.persistence.*;

@Data
@Entity
@Table(name = "t_area")
public class Area {

    @Id
    @GeneratedValue(generator = "uuid-generator")
    @GenericGenerator(name = "uuid-generator", strategy = "uuid")
    private String id;

    @Column(name = "name")
    private String name;

    @Column(name = "parent_id")
    private String parentId;

    @Column(name = "code")
    private String code;
}
```

JPA 自动生成的 DDL：

```sql
-- auto-generated definition
create table t_area
(
    id        varchar(255) not null
        primary key,
    code      varchar(255) null,
    name      varchar(255) null,
    parent_id varchar(255) null
);
```

---

## 邻接表

这是最常见的一种存储树结构的方式，每个元素节点持有一个字段，该字段保存着父节点的 id，在本文中，我将其命名为 parent_id。

例如，一个简单的行政区划表：

|  id  |  name  |  code  | parent_id |
| :--: | :----: | :----: | :-------: |
|  1   |  中国  |        |    -1     |
|  2   | 浙江省 | 330000 |     1     |
|  3   | 江苏省 | 320000 |     1     |
|  4   | 宁波市 | 330200 |     2     |
|  5   | 杭州市 | 330100 |     2     |

首先我们需要明晰一些术语，父节点，字节点，叶子节点，祖先节点和后代节点：

在这 5 条数据中，浙江省和江苏省的 parent_id 都为 1，所以中国作为这两个省的父节点（parent node），这两个省就是中国的子节点（children node）。

同样的，宁波市和杭州市的 parent_id 都为 2，所以它们都属于浙江省的子节点。

中国是其他所有结点的祖先节点（ancestors），其他所有节点都是中国的后代（descendants）。

宁波市和杭州市在该表中没有后代，所以它们被称为叶子节点（leaf node）。

### 带有子节点引用列表的 VO 类

我们假设 MySQL 中已经存在了像上面一样的表和数据，我们需要构造一棵完整的树返回给前端。

拿到数据库中的原始数据很简单：

```java
List<Area> areas = areaRepository.findAll();
```

现在需要做的就是将扁平的 area list 转成树结构，为此需要另外编写一个带有子节点列表的类，返回给前端：

AreaVO.java

```java
import lombok.Data;

import java.util.List;

@Data
public class AreaVO {

    private String id;

    private String name;

    private String parentId;

    private String code;

    private List<AreaVO> children;
}
```

AreaVO 和 Area 不同点在于，AreaVO 作为返回给前端的对象，比面向数据库的 Area 多了一个 children 字段，里面存储着下一级的子节点的引用。

在构造 children 字段前，我们可以使用 BeanUtils.copyProperties()，将除了 children 之外的字段，从原始 Area 复制到 AreaVO 中。

```java
// 从数据库中提取的原始数据
List<Area> areas = areaRepository.findAll();
// 返回给前端的类，包含 children 字段，可以被构造成完整的树结构
List<AreaVO> vos = new ArrayList<>();
// 将业务字段值拷贝到 VO 类中
areas.forEach(area -> {
    AreaVO vo = new AreaVO();
    BeanUtils.copyProperties(area, vo);
    vos.add(vo);
});
```

### 利用队列构造树结构

观察可以发现，每个节点可能即是父节点，也可能是子节点，根节点和叶子节点是例外，因外根节点只有子节点，没有父节点，而叶子节点只有父节点，没有子节点。

可以使用一种自顶向下，逐层构造的方式，最开始分成两部分，根节点和除了根节点之外的后代节点。

根节点去遍历后代节点，找到自己的直系后代（即，子节点），判断某个节点是不是自己的子节点很简单：

```java
// 如果 children 的 parentId 字段和当前父节点的 id 字段值相同，说明该 children 是 parent 的子节点之一
if (Objects.equals(childrenNode.getParentId(), parentNode.getId())) {
    parentNode.getChildren().add(childrenNode);
}
```

第一轮应该是根节点遍历找到所有自己的直接子节点，然后根节点就完成了自己的任务，它 children 字段中应该有了一些子节点的引用。

举个具体的例子，当“中国”开始寻找子节点时，第一轮遍历，它的 children 应该是北京市，天津市，河北省，山西省...，而杭州市，宁波市则不在它的 children list 当中。

而北京市，天津市，河北省，山西省...这些二级节点（我们姑且将中国的层级定为 1），它们也需要找到自己的 children（省下面是市）。

所以很自然的，想到了队列这个数据结构，一开始“中国”节点进入队列，当它找到所有子节点之后，出队列，中国的子节点们进入队列，开始寻找它们的子节点，以此类推，直到整个队列为空，说明没有节点需要找子节点了，整个循环结束。

下面是我的方法，过程很简单，基本实现了上面的思路：

```java
private void treeHelper(Queue<AreaVO> parentNodes, List<AreaVO> childrenNodes) {
        // 循环终止条件是队列为空
        while (parentNodes.size() > 0) {
            // 取出队列中首个父节点
            AreaVO parentNode = parentNodes.poll();

            // 迭代遍历子节点列表，寻找当前父节点下的子节点
            // 这里使用迭代器遍历子节点列表，是因为循环中涉及到了 remove 的操作
            Iterator<AreaVO> iterator = childrenNodes.iterator();
            while (iterator.hasNext()) {
                AreaVO childrenNode = iterator.next();
                // 如果 children 的 parentId 字段和当前父节点的 id 字段值相同，说明该 children 是 parent 的子节点之一
                if (Objects.equals(childrenNode.getParentId(), parentNode.getId())) {
                    // 父节点的 list 可能为 null，表示是叶子节点
                    // 如果找到了子节点，就不再是叶子节点了，需要构造一个空的 list 存放子节点
                    if (parentNode.getChildren() == null) {
                        parentNode.setChildren(new ArrayList<>());
                    }
                    // 将 children 放到 parent 的 children list 中
                    parentNode.getChildren().add(childrenNode);
                    // 迭代器移除当前的子节点，如果不进行该操作的话，childrenNodes 的大小永远不变，会增加后面循环的时间开销
                    // 因为每个节点只可能有一个父节点，children 找到了自己的父节点之后，它就不可能再是别人的子节点了，所以
                    // 可以直接从 childrenNodes 中移除，每次循环，childrenNodes 都会少一些，增加后面循环的效率
                    iterator.remove();
                    // 当前子节点，也可能是别的节点的父节点，所以需要进入队列
                    parentNodes.add(childrenNode);
                }
            }
        }
    }
```

完整的 service 层代码

```java
import cn.korilweb.demoadministrativedivisions.model.entity.Area;
import cn.korilweb.demoadministrativedivisions.model.vo.AreaVO;
import cn.korilweb.demoadministrativedivisions.repository.AreaRepository;
import cn.korilweb.demoadministrativedivisions.service.AreaService;
import org.springframework.beans.BeanUtils;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Service;

import java.util.*;
import java.util.function.Predicate;
import java.util.stream.Collectors;

@Service
public class AreaServiceImpl implements AreaService {

    @Autowired
    private AreaRepository areaRepository;

    @Override
    public AreaVO areaTree(String id) {
        // 数据库原始数据
        List<Area> areas = areaRepository.findAll();
        // 返回给前端树形结构 VO 类
        List<AreaVO> vos = new ArrayList<>();
        // 拷贝业务字段值
        areas.forEach(area -> {
            AreaVO vo = new AreaVO();
            BeanUtils.copyProperties(area, vo);
            vos.add(vo);
        });

        // 找到根节点，约定 id = 0 是根节点
        Predicate<AreaVO> isRoot = areaVO -> Objects.equals(areaVO.getId(), "0");
        
        Queue<AreaVO> parentNodes = new ArrayDeque<>();
        // 将根节点放到队列中
        AreaVO rootNode = vos.stream().filter(isRoot).findFirst().get();
        parentNodes.offer(rootNode);
        // 除了根节点之外的所有子节点
        List<AreaVO> childrenNodes = vos.stream().filter(isRoot.negate()).collect(Collectors.toList());
        // 构造树方法
        treeHelper(parentNodes, childrenNodes);
        return rootNode;
    }

    private void treeHelper(Queue<AreaVO> parentNodes, List<AreaVO> childrenNodes) {
        // 循环终止条件是队列为空
        while (parentNodes.size() > 0) {
            // 取出队列中首个父节点
            AreaVO parentNode = parentNodes.poll();

            // 迭代遍历子节点列表，寻找当前父节点下的子节点
            Iterator<AreaVO> iterator = childrenNodes.iterator();
            while (iterator.hasNext()) {
                AreaVO childrenNode = iterator.next();
                // 如果 children 的 parentId 字段和当前父节点的 id 字段值相同，说明该 children 是 parent 的子节点之一
                if (Objects.equals(childrenNode.getParentId(), parentNode.getId())) {
                    // 父节点的 list 可能为 null，表示是叶子节点
                    // 如果找到了子节点，就不再是叶子节点了，需要构造一个空的 list 存放子节点
                    if (parentNode.getChildren() == null) {
                        parentNode.setChildren(new ArrayList<>());
                    }
                    // 将 children 放到 parent 的 children list 中
                    parentNode.getChildren().add(childrenNode);
                    // 迭代器移除当前的子节点，如果不进行该操作的话，childrenNodes 的大小永远不变，会增加后面循环的时间开销
                    // 因为每个节点只可能有一个父节点，children 找到了自己的父节点之后，它就不可能再是别人的子节点了，所以
                    // 可以直接从 childrenNodes 中移除，每次循环，childrenNodes 都会少一些，增加后面循环的效率
                    iterator.remove();
                    // 当前子节点，也可能是别的节点的父节点，所以需要进入队列
                    parentNodes.add(childrenNode);
                }
            }
        }
    }
}
```

可以看到 service 方法中，我额外放了一个 id 字段，这是为了应付这样的需求：前端不想要一棵以中国为根节点的树，而是需要一颗以杭州市为根节点的树。

理想情况就是，前端传一个它所期望的根节点的 id，我们后端返回以该 id 的节点为根节点构造的一棵树。

```java
// 找到根节点，约定 id = 0 是根节点
Predicate<AreaVO> isRoot = areaVO -> Objects.equals(areaVO.getId(), "0");
```

只需要把上面的 isRoot 换成前端传来的 id 值即可。

```java
// 找到根节点，根据前端传来的 id 值判断
Predicate<AreaVO> isRoot = areaVO -> Objects.equals(areaVO.getId(), id);
```

这种方式有个很大的缺点，比如前端传来一个杭州市的 id 值，那么杭州市的父节点们也会进入 childrenNodes 中，而且在之后的每一次遍历都不会被删除。

我能想到的一个简单的办法是，在数据库表中增加一个字段——level，事实上很多实现也是这么做的，它表示该节点所处的层级，中国是1 级，leve = 1，直辖市，省，自治区，特别行政区在 2 级，level = 2，以此类推。

如果传来杭州市，杭州市的 level = 3，在数据库查询的时候，就可以通过 leve 把杭州市的父节点们都排除，只查询出 level > 3 的节点。

### 利用递归构造树结构



---

## 树节点的新增操作



---

## 树节点的更新操作



---

## 树节点的删除操作



## 节点的排序



---

## 参考

1. https://mp.weixin.qq.com/s/tJ5m4BWYeZEiCjbxtwN1qA
2. https://mp.weixin.qq.com/s/yzy1EbzSS5Pb7boQ-jgPFw
3. https://mp.weixin.qq.com/s/sRivTVFKYYTzKXh9dQv1mw
