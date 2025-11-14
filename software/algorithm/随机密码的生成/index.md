---
title: "随机密码的生成"
date: 2025-11-14T11:11:08
summary: ""
---

## 前言

在服务器管理过程中，需要使用一些随机的密码，Linux 本身提供了一些命令（openssl或者urandom）提供了某种随机性的机制来生成密码，但是不够直观。

考虑到我运维的机器都有 Python，所以本文用一个 Python 脚本来生成随机密码。

## 实现

一般强度高一些的密码是 8-12 位，包含大小写字母，数字和特殊符号。

### 简单版本

```python
import string, random

length = 8
password = ''.join(random.choice(string.ascii_letters + string.digits + '!@#$') for _ in range(length))
```

这种适用于临时使用，它虽然包含了大小写、数字和指定的特殊字符，但是没有保证至少包含一个大写字母和一个特殊字符。

### 改进

为了保证至少 1 个大写字母和特殊字符，可以在 password_chars 这个 list 中，先加上一个随机的大写字母和特殊字符，剩下的部分在从字符集里面随机挑选，最后使用 random 的 shuffle 函数“洗牌”。

```python
import string, random

def gen_password(total_length: int = 8):

	if not (6 <= total_length <= 15):
		raise ValueError("密码长度必须在 6 到 15 之间")

	symbol = '!@#$'
	char_pool = string.ascii_letters + string.digits + symbol

	password_chars = []
	password_chars.append(random.choice(string.ascii_uppercase))
	password_chars.append(random.choice(symbol))

	remain_length = total_length - len(password_chars)

	for _ in range(remain_length):
		password_chars.append(random.choice(char_pool))

	random.shuffle(password_chars)
	return ''.join(password_chars)

```

运行效果如下：

```
>>> from test import gen_password
>>> [gen_password(8) for _ in range(10)]
['KCh1jI3!', 'qaDYWMg!', 'PqgS$@RP', 's8@xMV#B', 'oW5Tv@gw', '$naVJN@!', 'nzjHD$F8', 'IGS$DcYd', 'lOWi8V$u', 'ONInci#P']
>>>
```
