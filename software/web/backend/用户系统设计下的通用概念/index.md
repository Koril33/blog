---
title: "用户系统设计下的通用概念"
date: 2025-12-12T11:24:20
summary: "独立于各个 Auth 框架之外的抽象概念"
---

## 前言

在所有涉及安全性的系统中，用户登录、注册相关的功能在许多概念上，具有独立于框架或系统的部分。

系统为了保证资源能够被正确的人在正确的时间以及正确的地点访问，无论这个系统是操作系统（Linux、Windows）、社交平台（Youtube、X、Instagram）、数据库管理系统（PostgreSQL、MySQL），亦或者是 web 框架（Spring Security，Flask-Security，Casbin，Laravel Auth），它们都需要提供完整的认证和授权体系，这种体系几乎独立的存在于所有系统内，也就是用户系统。

本文简单的概述下这些用户系统在概念上的通用部分。

---

## 身份-Identity

无论客户端以怎样的方式访问系统，系统都需要通过“身份”（Identity）来标记这次访问是由谁发起的。

这种标记建立在该 Identity 已经存在于系统中，如果不存在，则需要系统提供注册的跳转或者拒绝访问的提示。

访问系统的客户端之所以用 Identity 而不是 User，是因为访问系统的不一定是人类，也可以是某台物联网设备。

---

## 认证-Authentication

假设客户端提供的 Identity 确实存在于系统之中，那么第二步就是要确认这个 Identity 真的属于当前访问的客户端（Identity Verification），而不是客户端冒名顶替或者伪造。

常见的方式：

- 密码认证
- 短信、邮箱验证码
- 邮箱链接
- OAuth2
- 生物认证
- TOTP

这些技术只有一个目的，客户端给定一个 Identity，通过以上方式的某一种来证明 Identity 属于自己。

---

## 凭证-Credential

上面说的 Authentication 指的是认证 Identity 的流程，在这个流程中使用的用于证明身份的元素，被称为凭证。

常见的凭证：

- 普通的密码（一般会使用 Hash 算法加密）
- 2FA secret
- OAuth2 Token
- Recovery codes

凭证是在认证过程中使用到的数据。

---

## 授权-Authorization

假设客户端提供的 Identity 存在于系统，并且通过持有的凭证成功完成了认证的流程，接下来就是要判断该 Identity 可以访问哪些资源，可以完成哪些操作。

一个低级别的员工和一个董事会成员，他们俩持有的账号，登陆相同的公司系统，能看到的信息肯定是不一样的，因为后者的权限更多，这里就涉及到授权。

授权模型（或者叫访问控制模型 Access Control Models）有很多种，以下列举一些常见的授权模型。

### DAC

DAC（Discretionary Access Control），自主访问控制，由资源的持有者来决定其他用户是否有权限访问资源。

DAC 通过 ACLs（Access Control Lists）来决定谁有权限访问，ACL 一般是一张表，记录了 Identity 和其拥有的权限，比如：

```
Bob:READ,WRITE
Tom:READ
```

表示对于该资源，Bob 可读可写，Tom 仅可读不可写。

DAC 的优势在于灵活和简单直观，它把权限配置的功能分散在了各个资源持有者身上，但是缺点就是很难从全局把控系统某个用户的权限（比如，如果想知道 Bob 的所有权限，得查看所有的 ACLs）。

DAC 一般用在文件系统（File System）或者网络资源上的权限分配。

### MAC

MAC（Mandatory Access Control），强制访问控制，和分散式的 DAC 相反，系统统一维护安全级别，用户不能随意修改权限，一般 MAC 通过对资源和用户打上某种标签，来决定哪些用户可以访问哪些资源。

MAC 起源于军事和情报界，给一些文档打上了安全等级标签（公开 < 秘密 < 机密 < 绝密），不同级别的人可以不同级别的文档。

MAC 严格的访问规则系统会根据主体和客体的安全标签，强制判定访问权限，主体无权自主授予或撤销权限，所有访问决策由系统管理员或预设的安全策略决定。

在国家安全领域之外，银行和保险公司可能会使用 MAC 来控制对客户账户数据的访问。

### RuBAC

RuBAC（Rule-Based Access Control），基于规则的访问控制，RuBAC 的规则非常灵活，可以适用于那种访问场景比较动态的业务，常见的规则如下：

- 仅允许用户在某一天的某个时间段内访问资源
- 限制允许访问的设备类型
- 限制允许访问的地理位置和网络类型

RuBAC 会在每次用户访问系统的时候，检查这些预定义的规则，所以非常动态化。

### RBAC

RBAC（Role-Based Access Control），基于角色的访问控制，在用户量比较庞大的企业中，常常使用角色来对用户进行分类，一个用户可以拥有多个角色。

RBAC 是非常常用的模型，因为它的配置很直观，将对资源访问权限的配置交给了角色，这样配置好了一个角色的权限后，拥有该角色的用户群体将自动继承这些权限。

---

## 用户信息-UserProfile

用户信息往往是一些用户的元数据，它们和整个认证-授权系统无关，只是表示用户本身的数据。

- 昵称
- 头像
- 生日
- 国家

这些信息不参与认证，实践过程中，可以把这些信息和认证信息分开存放。

---

## 资源-Resource

受保护的资源，可以是文件，API 接口，数据等等。

---

## 生命周期-Lifecycle

用户从未注册到删除账号，完整的生命周期：

- 注册
- 验证注册用户的手机号或者邮箱
- 登录
- 忘记密码
- 账号锁定（违反风控）
- 注销
- 删除

---

## 参考

1. https://www.permit.io/blog/planning-authorization-model-and-architecture-full-2025-guide
2. https://www.twingate.com/blog/other/access-control-models
3. https://developer.huawei.com/consumer/cn/blog/topic/03200764394012152
4. https://www.acresecurity.com/blog/rule-based-access-control-rubac-the-complete-guide
5. https://auth0.com/intro-to-iam
6. https://www.geeksforgeeks.org/system-design/designing-authentication-system-system-design/
7. https://www.cerbos.dev/blog/designing-an-authorization-model-for-an-enterprise