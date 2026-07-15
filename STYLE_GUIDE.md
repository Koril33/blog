# 规范指导

## 文章修改

### 文章和目录结构

发现标题不符合的，你可以修改文章标题，index.md 里的元数据：

```

---
title: "Python的Thread基础"
date: 2026-01-01T09:35:51
summary: "并发编程中的基本工具"
---

```

title 要和 index.md 所属目录的名字一致。

文章元数据没有 summary 的，帮我补充 summary。

如果发现可以单独形成目录的，你可以新增目录，然后把文章放进新的目录里。比如：software\web\backend 下面包含大量 SpringSecurity 的内容，你可以新建一个目录，把相关文章放进去。

整理所有目录，确保层级不要太深，分类也可以整理下，把有些不符合此类的文章从目录中移动到相应的目录中。

### 语言

- 保留作者个人语气
- 不改写作者观点
- 不增加不存在的信息
- 中文使用全角标点
- 英文单词前后留空格
- 句子中的代码词汇使用 `` 来包裹
- 技术术语大写开头，使用：Git 而不是 git
- 不要修改代码逻辑。
- 链接使用 []() 来包裹
- 其他符合 markdown 语法规范也可以修改
- 部分遗漏 TOC 标记文章加上 TOC 标记

### 代码

用 ``` 包裹的代码格式化，去除不必要的缩进。

### 参考

你可以参考一下写作“圣经”

1. https://zh-style-guide.readthedocs.io/zh-cn/latest/index.html
2. https://learn.microsoft.com/zh-cn/style-guide/welcome/
3. https://developers.google.com/style/

---

## 网站

所有 markdown 最后都会转换成 HTML，由我自己编写的 CLI 工具进行转换，github 地址：https://github.com/Koril33/blogger

具体的转换代码位于：https://github.com/Koril33/blogger/blob/main/src/djhx_blogger/gen.py

你需要确保你修正的 markdown 能够经过脚本转换正常显示。

最终生成的网站：https://blog.djhx.site/

---

## 其他

所有修改的文章，每一篇改了哪些内容，记录成一个 CHANGE.md。

忽略 about 和 cook 目录。