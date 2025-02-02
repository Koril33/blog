---
title: "SpringMVC接收前端参数的N种方式"
date: 2023-02-12T18:23:56+08:00
tags: []
featured_image: "images/background.jpg"
summary: "后端和前端对接口时接收参数的问题，本文一口气把所有可能的情况都罗列出来"
toc: true
---

## 前言

面试官们见了 SpringMVC，也每每这样问他，引人发笑。SpringMVC 自己知道不能和他们谈天，便只好向孩子说话。有一回对我说道，“你写过 web 接口么？”我略略点一点头。他说，“写过接口，……我便考你一考。后端接受前端参数，怎样收的？”我想，讨饭一样的人，也配考我么？便回过脸去，不再理会。SpringMVC 等了许久，很恳切的说道，“不能写罢？……我教给你，记着！这些注解应该记着。将来做高级码农的时候，写接口要用。”我暗想我和高级码农的等级还很远呢；又好笑，又不耐烦，懒懒的答他道，“谁要你教，不就是 @RequestMapping 加个方法么？”SpringMVC 显出极高兴的样子，将两个指头的长指甲敲着柜台，点头说，“对呀对呀！……除了这个注解还有其他的，你知道么？”我愈不耐烦了，努着嘴走远。SpringMVC 刚用指甲蘸了酒，想在柜上写字，见我毫不热心，便又叹一口气，显出极惋惜的样子。

就像是茴香豆的茴字有四种写法一样，工作里前后端接口对接，总的来说都是通过 HTTP 协议，前端传递字符串或者特定格式（比如 JSON 或者 XML）的信息，后端如果用的是 SpringMVC 框架，一般会用到一些注解，本文简单的记录下工作里常用的一些方式，做个备忘。

---

## 涉及的基本类

User.java，用于存储用户信息：

```java
import lombok.Data;

@Data
public class User {

    private String username;
    private String password;
}
```

UserController.java：

```java
@RestController
@RequestMapping("/api")
public class UserController {
	// 后续的接口代码
    // ...
}
```

---

## URL 加请求参数

有时候请求一些页面需要加一些简单的，不敏感的参数，比如分页信息，传输页面编号和数量，又比如博客网站，需要传递一个文章编号，这些简单的信息可以用 GET 请求，把参数加在 URL 后面，比如豆瓣的一个链接：https://movie.douban.com/top250?start=125&filter=，里面有两个参数一个是 start，另一个是 filter，中间用 & 符号分割。

### HttpServletRequest 获取参数

```java
// GET
// http://localhost:8080/api/getInfo1?username=djh&password=123
@GetMapping("/getInfo1")
public String getInfo1(HttpServletRequest request) {
    String username = request.getParameter("username");
    String password = request.getParameter("password");

    System.out.println("getInfo1");
    System.out.println("username: " + username);
    System.out.println("password: " + password);

    return "ok";
}
```

### 直接映射

如果前端传来的 URL 参数和后端方法形参一致，则可以直接映射：

```java
// GET
// http://localhost:8080/api/getInfo2?username=djh&password=123
@GetMapping("getInfo2")
public String getInfo2(String username, String password) {

    System.out.println("getInfo2");
    System.out.println("username: " + username);
    System.out.println("password: " + password);

    return "ok";
}
```

### @RequestParam

如果前端传来的 URL 参数和后端方法的形参不一致，比如前端传来的是 name 和 pwd，而后端方法里的参数名为 username 和 password，则须使用 @RequestParam 来手动映射：

```java
// GET
// http://localhost:8080/api/getInfo3?name=djh&pwd=123
@GetMapping("getInfo3")
public String getInfo3(@RequestParam("name") String username,
                       @RequestParam("pwd") String password) {

    System.out.println("getInfo3");
    System.out.println("username: " + username);
    System.out.println("password: " + password);
    return "ok";
}
```

### 映射成对象

如果前端 URL 参数的数量太多了，后端方法形参就显得臃肿不堪，可以把这些参数放进对象中：

```java
// GET
// http://localhost:8080/api/getInfo4?username=djh&password=123
@GetMapping("getInfo4")
public String getInfo4(User user) {
    System.out.println("getInfo4");
    System.out.println(user);
    return "ok";
}
```

---

## URL 路径参数

路径作为参数，比如：http://myblog/post/123，这里 post 和 123 作为路径参数来查询信息，这时候用 @PathVariable 映射即可：

```java
// GET
// http://localhost:8080/api/getInfo5/djh/123
@GetMapping("/getInfo5/{username}/{password}")
public String getInfo5(@PathVariable String username, @PathVariable String password) {
    System.out.println("getInfo5");
    System.out.println("username: " + username);
    System.out.println("password: " + password);
    return "ok";
}
```

---

## form-data 和 x-www-form-urlencoded 的区别

关于这两者的差别和使用场景，参考下面两个链接：

* https://www.baeldung.com/postman-form-data-raw-x-www-form-urlencoded
* https://stackoverflow.com/questions/26723467/postman-chrome-what-is-the-difference-between-form-data-x-www-form-urlencoded

## form-data

```java

// POST
// form-data(username=djh, password=123)
// http://localhost:8080/api/getInfo6
@PostMapping("/getInfo6")
public String getInfo6(String username, String password) {
    System.out.println("getInfo6");
    System.out.println("username: " + username);
    System.out.println("password: " + password);
    return "ok";
}

// POST
// form-data(name=djh, pwd=123)
// http://localhost:8080/api/getInfo7
@PostMapping("/getInfo7")
public String getInfo7(@RequestParam("name") String username,
                       @RequestParam("pwd") String password) {
    System.out.println("getInfo7");
    System.out.println("username: " + username);
    System.out.println("password: " + password);
    return "ok";
}

// POST
// form-data(username=djh, password=123)
// http://localhost:8080/api/getInfo8
@PostMapping("/getInfo8")
public String getInfo8(User user) {
    System.out.println("getInfo8");
    System.out.println(user);
    return "ok";
}
```

---

## x-www-form-urlencoded

```java
// POST
// x-www-form-urlencoded(username=djh, password=123)
// http://localhost:8080/api/getInfo9
@PostMapping("/getInfo9")
public String getInfo9(String username, String password) {
    System.out.println("getInfo9");
    System.out.println("username: " + username);
    System.out.println("password: " + password);
    return "ok";
}

// POST
// x-www-form-urlencoded(name=djh, pwd=123)
// http://localhost:8080/api/getInfo10
@PostMapping("/getInfo10")
public String getInfo10(@RequestParam("name") String username,
                        @RequestParam("pwd") String password) {
    System.out.println("getInfo10");
    System.out.println("username: " + username);
    System.out.println("password: " + password);
    return "ok";
}

// POST
// x-www-form-urlencoded(username=djh, password=123)
// http://localhost:8080/api/getInfo11
@PostMapping("/getInfo11")
public String getInfo11(User user) {
    System.out.println("getInfo11");
    System.out.println(user);
    return "ok";
}
```

---

## raw

### text

```java
// POST
// raw-text(username=djh, password=123)
// http://localhost:8080/api/getInfo12
@PostMapping("/getInfo12")
public String getInfo12(@RequestBody String text) {
    System.out.println("getInfo12");
    System.out.println(text);
    return "ok";
}
```

### json

```java
// POST
// raw-json(
// {
//      "username": "djh",
//      "password": "123"
// }
// )
// http://localhost:8080/api/getInfo13
@PostMapping("/getInfo13")
public String getInfo13(@RequestBody User user) {
    System.out.println("getInfo13");
    System.out.println(user);
    return "ok";
}
```

### xml

xml 比较特殊，需要额外在 User.java 上面加个 @XmlRootElement 注解：

```java
import jakarta.xml.bind.annotation.XmlRootElement;
import lombok.Data;

@Data
@XmlRootElement(name = "xml-user")
public class User {

    private String username;
    private String password;
}
```

maven 需要额外引入两个依赖：

```xml
<dependency>
    <groupId>jakarta.xml.bind</groupId>
    <artifactId>jakarta.xml.bind-api</artifactId>
</dependency>

<dependency>
    <groupId>org.glassfish.jaxb</groupId>
    <artifactId>jaxb-runtime</artifactId>
</dependency>
```

controller 的代码没有什么变化：

```java
// POST
// raw-xml(
// <xml-user>
//    <username>djh</username>
//    <password>123</password>
// </xml-user>
// )
// http://localhost:8080/api/getInfo14
@PostMapping("/getInfo14")
public String getInfo14(@RequestBody User user) {
    System.out.println("getInfo14");
    System.out.println(user);
    return "ok";
}
```

---

## 多个对象

上面的例子都是单个 User 对象信息，下面简单介绍一下多个对象接受的方法。

### List

```java
// POST
// raw-json(
// [
//    {
//        "username": "djh",
//            "password": "123"
//    },
//    {
//        "username": "lzq",
//        "password": "456"
//    }
// ]
// )
// http://localhost:8080/api/getInfo15
@PostMapping("/getInfo15")
public String getInfo15(@RequestBody List<User> userList) {
    System.out.println("getInfo15");
    userList.forEach(System.out::println);
    return "ok";
}
```

### Map

map 可以用作单个对象，但是这对后期项目维护不太友好，如果你是用 User 对象接受前端参数，那么后期维护的程序员一眼就能看出，前端传了什么参数的对象过来，但是如果改成 Map，只能靠 log 打印或者求助于文档了，没法直接从源码看出前端传了什么。

```java
// POST
// raw-json(
// {
//     "username": "djh",
//     "password": "123"
// }
// )
// http://localhost:8080/api/getInfo16
@PostMapping("/getInfo16")
// 单纯的一个 Map<String, String>，无法直接看出传递了什么参数
public String getInfo16(@RequestBody Map<String, String> userMap) {
    System.out.println("getInfo16");
    userMap.forEach((k, v) -> System.out.println("key: " + k + ", value: " + v));
    return "ok";
}
```

Map 对象用作多个参数：

```java
// POST
// raw-json(
// {
//     "1": {
//          "username": "djh",
//          "password": "123"
//      },
//
//     "2": {
//          "username": "lzq",
//          "password": "456"
//      }
// }
// )
// http://localhost:8080/api/getInfo17
@PostMapping("/getInfo17")
// 这里至少知道值是 User，至于键的语义是什么，无法一眼看出，只能知道键的类型是个 String
public String getInfo17(@RequestBody Map<String, User> userListMap) {
    System.out.println("getInfo17");
    userListMap.forEach((k, v) -> System.out.println("key: " + k + ", value: " + v));
    return "ok";
}
```

---

## 一个稍微有些复杂的对象

上面的 User.java 只有两个属性，工作的时候应付的数据模型一般更加复杂，这里用以下这个订单类来做个示例：

Order.java

```java
import lombok.Data;

import java.math.BigDecimal;
import java.time.LocalDateTime;
import java.util.List;

@Data
public class Order {
	// 订单编号
    private String orderId;
	// 下单时间
    private LocalDateTime orderDateTime;
	// 用户信息
    private ClientInfo clientInfo;
	// 商品列表
    private List<Item> items;
	// 是否已经付钱了
    private boolean isPaid;
	// 总价
    private BigDecimal totalPrices;
	// 订单备注信息
    private String remark;
}
```

Order 中的用户信息类，ClientInfo.java

```java
import lombok.Data;

@Data
public class ClientInfo {
	// 客户编号
    private String clientId;
	// 客户名称
    private String clientName;
	// 地址信息
    private Address address;
}
```

ClientInfo 中的地址信息类，Address.java

```java
import lombok.Data;

@Data
public class Address {
	// 国家
    private String country;
	// 省份
    private String province;
	// 城市
    private String city;
	// 区
    private String district;
	// 地址详细信息
    private String addressDetail;
}
```

Order 中的物品类，Item.java

```java
import lombok.Data;

@Data
public class Item {
	// 商品编号
    private String itemId;
	// 商品名称
    private String itemName;
}
```

Controller：

```java
@PostMapping("/getInfo18")
public String getInfo18(@RequestBody List<Order> orderList) {
    System.out.println("getInfo18");
    orderList.forEach(System.out::println);
    return "ok";
}
```

前端传递一个订单列表的 JSON 信息即可：

orderList.json

```json
[
    {
        "orderId": "21783912783",
        "orderDateTime": "2023-01-14T13:42:11",
        "clientInfo": {
            "clientId": "123",
            "clientName": "Tom",
            "address": {
                "country": "中国",
                "province": "浙江省",
                "city": "杭州市",
                "district": null,
                "addressDetail": ""
            }
        },
        "items": [
            {
                "itemId": "191991",
                "itemName": "被子"
            },
            {
                "itemId": "111111",
                "itemName": "桌子"
            },
            {
                "itemId": "789798",
                "itemName": "椅子"
            }
        ],
        "isPaid": true,
        "totalPrices": "128.45",
        "remark": null
    },

    {
        "orderId": "11283942553",
        "orderDateTime": "2023-01-13T18:42:11",
        "clientInfo": {
            "clientId": "111",
            "clientName": "Rick",
            "address": {
                "country": "中国",
                "province": "浙江省",
                "city": "杭州市",
                "district": "滨江区",
                "addressDetail": "xxx街道"
            }
        },
        "items": [
            {
                "itemId": "8282828",
                "itemName": "杯子"
            }
        ],
        "isPaid": false,
        "totalPrices": "20.00",
        "remark": "this is a remark"
    }

]
```
