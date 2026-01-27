---
title: "角色、用户和权限"
date: 2026-01-19T15:01:21
summary: "管理 PG 数据库的基本功能"
---

## 角色和用户

从 PG 的 8.1 版本之后，PG 只有角色（Role）这一个概念，用户（User）和组（Group）实际上是特殊的角色，用户可以看作是拥有登录数据库权限的角色。

### 默认超级用户

安装好 PG 后，默认有一个超级角色或者说是超级用户——postgres，它相当于 Linux 里的 root，拥有数据库的最高权限，一般不会直接使用，而是通过 postgres 来创建其他权限较小的角色或者用户来使用 PG。

除了 postgresql 整个默认用户外，还有其他的一些默认角色：

```sql

postgres=# SELECT rolname FROM pg_roles;
          rolname          
---------------------------
 postgres
 pg_database_owner
 pg_read_all_data
 pg_write_all_data
 pg_monitor
 pg_read_all_settings
 pg_read_all_stats
 pg_stat_scan_tables
 pg_read_server_files
 pg_write_server_files
 pg_execute_server_program
 pg_signal_backend
 pg_checkpoint

```

PG 的超级用户 postgres 同时会在 Linux 下设置一个对应的 nologin 用户，名字也叫 postgres。

所以在安装完 PG 后，我们通常会通过`sudo -iu postgres`来切换当前Linux用户，以获得 postgres 的权限，然后通过`psql`无密码直接登录数据库系统。

之所以能这样做，是因为 hba 文件的缘故，在`pg_hba.conf`中的第一条规则：

```
# DO NOT DISABLE!
# If you change this first entry you will need to make sure that the
# database superuser can access the database using some other method.
# Noninteractive access to all databases is required during automatic
# maintenance (custom daily cronjobs, replication, and similar tasks).
#
# Database administrative login by Unix domain socket
local   all             postgres                                peer
```

该规则表示使用 Unix socket 连接（local）的情况下，postgres 角色使用的认证方式是 peer。

peer 的认证逻辑是如果当前 OS 用户名等同于数据库用户名，那么无需密码直接通过。

对于自己创建的角色/用户也是一样的，因为 hba 的第二条规则是：

```
# "local" is for Unix domain socket connections only
local   all             all                                     peer
```

这里的 all 匹配了所有的用户名，所以如果你登陆 Linux 的用户名和 PG 的用户名一致，那么就可以不用密码直接登录到数据库。

### public 角色

PG 包含一个默认的角色——PUBLIC。


### 创建和删除角色

用 SQL 语句创建和删除角色：
```sql
CREATE ROLE name;
DROP ROLE name;
```
命令行工具创建和删除角色：
```shell
createuser name
dropuser name
```

创建一个可以登录的角色：
```sql
CREATE ROLE name LOGIN;
CREATE USER name;
```

拥有 LOGIN 属性的角色可以被视为数据库用户（database user），并且能够通过该用户连接数据库。

### 角色属性

角色的属性决定了该角色拥有哪些操作数据库的权力，除了上面已经提到的 LOGIN 之外，还有一下一些属性：

- SUPERUSER
- CREATEDB
- CREATEROLE
- REPLICATION
- PASSWORD
- INHERIT
- BYPASSRLS
- CONNECTION LIMIT

---

## 组和角色

### 组的概念

之前提到角色可以看作是包含了登录权限的用户，也可以是组（Group），组的概念就是用于解耦权限分配和用户。

我们可以为一个角色分配一些权限，然后把拥有这些权限的用户聚合到这个角色下，那么这个角色就可以称之为“组”。

把用户分组在一起来便于管理权限常常很方便：那样，权限可以被授予一整个组或从一整个组回收。

创建、删除组的语法和创建一个普通角色的语法一致：

```sql
CREATE ROLE group_name;
DROP ROLE group_name;
```

通常，作为组使用的角色不会包含登录属性，但如果您需要，也可以设置组含登录属性。

可以通过`GRANT`和`REVOKE`来把成员（Memeber，可以是角色，也可以是用户）添加到组或者从组中删除。

```sql
GRANT group_role TO role1, ... ;
REVOKE group_role FROM role1, ... ;
```

PG 不允许出现角色间的循环依赖（比如：A 属于 B，B 属于 A）这种情况。

PUBLIC 又是所有角色所属的默认根角色，所以你无法把 PUBLIC 添加到任何组内。

### 切断权限扩散

PG 为了避免一个用户在继承多个角色时，可能导致用户所拥有的权限过多的问题，通过`INHERIT`和`SET`的命令来起到切断权限扩散的作用。

有两种方式使得成员使用所属组的权限：

1. 显示的使用 SET ROLE 成为临时的组角色。
2. 如果在把成员添加到组的 SQL 中使用了 INHERIT TRUE，那么该角色会继承组的权限。

INHERIT 也有例外：角色属性 LOGIN、SUPERUSER、CREATEDB 和 CREATEROLE 可以被认为是一种特殊权限，它们从来不会像数据库对象上的普通权限那样被继承。

要使用这些特殊权限，你必须实际 SET ROLE 到一个有这些属性之一的特定角色。

SET ROLE 某个角色后，可以通过以下命令恢复到原始角色：

```sql
SET ROLE joe;
SET ROLE NONE;
RESET ROLE;
```

---

## 权限


---

## 参考

1. https://www.postgresql.org/docs/current/user-manag.html
2. https://www.postgresql.org/docs/current/ddl-priv.html
3. https://www.postgresql.org/docs/current/client-authentication.html
4. https://www.postgresql.org/docs/current/sql-grant.html