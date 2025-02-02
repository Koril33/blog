---
title: "自己动手写一个博客静态页面生成器"
date: 2025-01-29T09:30:00+08:00
summary: "使用 Python + HTML + CSS 完成一个简单的 markdown 转静态页面的脚本"
---

## 目录

[TOC]

---

## 前言

最早的时候，我的博客使用 [hexo](https://hexo.io/zh-cn/) 来作为框架，后来到了毕业的时候换成了 [hugo](https://gohugo.io/)，一用就是两三年。我很喜欢静态博客的原因是简单方便，没有数据库，没有后端语言的介入，纯 HTML + CSS，外加一点 JS，很多针对后端接口的漏洞攻击都无效了，并且静态页面的性能也很好，配合浏览器的页面缓存，基本都是秒开。

动态博客更加复杂，功能很丰富，可以支持登陆注册，评论回复，后台上传博客内容，图片等，还能监控博客信息，比如每日的流量。但动态博客涉及到了后端框架和数据库等等，复杂性大的多，尽管有很多优秀的动态博客框架简化了这些操作，但是我个人在博客上，更加倾向静态页面，我看过很多很多人的博客，大部分的博客的内容数量和流量都不大，用动态博客需要托管在能够安装数据库的服务器上，有些杀鸡用宰牛刀的感觉。

现在我自己动手打算写一个静态博客生成器，原因有以下几点：

1. 我希望自己的网站的所有功能都能自己实现，金窝银窝不如自己的狗窝，外面的框架功能再丰富，轮子写的再精致，都不如自己手搓来的有成就感。很多朋友都会来一句：不是有现成的么，干吗自己搞？如果是工作，我无法反驳，用业界标准框架是对自己，同事，老板，客户负责。但是，业余时间搓的玩具也用现成的，未免有些无趣。
2. 很多框架都基于一些既定的标准，想要个性化就必须翻文档，改起来很麻烦，而且一旦变了主题，有些东西的配置还需要重新修改，改的过程中就诞生了这样的想法：这么多时间去查文档，都能自己重新写一个了。
3. 实现我目前的功能并不难，可以花大概一两天的时间达到目标。

---

## 要求

### 简单

简单有两个方面，一个是代码简单，我希望能够用最精简的代码和依赖库，来实现我目前的需求。

另一个就是样式要简单，感觉已经过了想要花里胡哨装点自己的年纪，看的清楚（无广告的老年模式）对我而言比较重要，当然更多原因是我的前端学的太烂了，不如用这些大佬简陋的 UI 来作为自己的挡箭牌吧～

- Redis 作者 Salvatore Sanfilippo：https://www.antirez.com/
- PostgreSQL 贡献者 Bruce Momjian：http://momjian.us/main/blogs/
- MySQL 之父 Monty Widenius：http://monty-says.blogspot.com/
- Flask 作者 Armin Ronacher：https://lucumr.pocoo.org/
- Django 共同创始人 Jacob Kaplan-Moss：https://jacobian.org/
- Ruby on Rails 作者 David Heinemeier Hansson (DHH) ：https://dhh.dk/
- Donald Knuth (算法大师,《计算机程序设计艺术》作者) ：https://www-cs-faculty.stanford.edu/~knuth/
- Dijkstra 手稿：https://www.cs.utexas.edu/~EWD/
- C++ 之父 Bjarne Stroustrup：https://www.stroustrup.com/

就目前而言，我觉得重要的还是内容，能够经年累月的产出高质量的文章比花哨的样式更有价值。

### 方便

设定好博客原始 markdown 目录的位置之后，通过脚本，一键完成目标 html 目录的生成，再一键完成部署到指定服务器的指定位置。

尽可能的减少编写一篇新博客文章的心智负担。

### 性能

性能上，其实没有太多的要求，因为静态博客的生成无非就是文本的转换和目录的创建，生成几十篇博客的 html 根本耗不了多少时间，太多的奇技淫巧（比如缓存）在这里施展不开拳脚。

---

## 需求

1. 所有博客采用 markdown 格式编写，最后转换成 html 文件。
2. 支持文章内插入图片，一篇文章的文本文件和图片文件存放到一个目录下，方便后期归档。
3. 文章的元信息（文章标题，时间，简介），以 yaml 格式放在 markdown 的开头（参考了 hugo）。
4. 整个博客目录以树形结构组织。
5. 树形结构的树节点有两种类型：分类节点（category node）和文章节点（article node），它们俩在文件系统层面都是目录，区别是分类节点下存储的可能包含子分类目录或者文章目录，文章节点也是目录，其下仅能存储一个index.md（文章的markdown文件）以及一个 images（存储图片的目录）。
6. 分类节点和文章节点的样式要做区分。

---

## 目录树结构

博客的 markdown 目录，组织结构类似如下：

```
blog(根节点)
 |
 |--nginx(分类节点)
 |    |
 |    |-nginx配置(文章节点)
 |    |    |-index.md(文章的 markdown 文件)
 |    |    |-images(存放文章中引用图片的目录)
 |    |
 |    |-nginx入门(文章节点)
 |         |-index.md
 |         |-images
 |
 |--database(分类节点)
 |     |
 |     |-mysql(分类节点)
 |     |
 |     |-redis(分类节点)
 |         |
 |         |-redis配置(文章节点)
 |              |-index.md
 |              |-images
 
```

规则很简单：分类节点可以嵌套分类节点或者文章节点，但是文章节点仅能包含一个 index.md 和 images 目录。

生成的结果 html 目录，组织结构如下：

```
public(根节点)
 |
 |--index.html(内容包含public下的链接)
 |
 |--nginx(分类目录)
 |    |-index.html(内容包含nginx下的链接)
 |    |-nginx配置(文章目录)
 |    |    |-index.html(index.md转换得到的)
 |    |    |-images(存放图片的目录)
 |    |
 |    |-nginx入门(文章目录)
 |         |-index.html
 |         |-images
 |
 |--database(分类目录)
 |     |-index.html
 |     |-mysql(分类目录)
 |     |   |-index.html
 |     |
 |     |-redis(分类目录)
 |         |-redis配置(文章目录)
 |             |-index.html
 |             |-images
 |

```

每个目录下一定包含一个 index.html，如果是分类目录，index.html 则包含了指向这个目录下的子分类目录或者文章目录的连接，如果是文章目录，则 index.html 就是原来的 markdown 转换后的 html 文件。

---

## 依赖

脚本使用 Python3 编写，涉及的依赖如下：

- beautifulsoup4
- jinjia2
- markdown

其实这些依赖也可以通过自己编写的代码实现，但目前考虑先上线使用一段时间，后面在逐步用自己写的代码替换这些依赖涉及的功能。

---

## 实现

### 项目结构

除了 main.py 外，还需要 css 样式和 template html 模板文件，目录结构如下：

```
blogger
  |-main.py
  |-css
  |  |-article.css
  |  |-category.css
  |
  |-template
     |-article.html
     |-category.html
```

因为 index.html 总共有两种类型——文章和分类，所以这里分别有两个 css 和 html 模板文件。

### 主函数逻辑

整个过程，主要分成两部分：

1. 遍历源目录，生成一个节点树
2. 根据节点树，生成目标目录

```python

def main():
    blog_dir_path_str = '/home/koril/project/djhx.site/blog'
    destination_blog_dir_name = 'public'
    root_node = walk_dir(blog_dir_path_str, destination_blog_dir_name)
    gen_blog_dir(root_node)
    cp_css(blog_dir_path_str)


if __name__ == '__main__':
    main()

```

### node 类

可以把源目录下的每个目录抽象成一个 node 节点，构造成一个树结构。

```python


class Node:
    cache_map = {}
    def __init__(self, source_path, destination_path, node_type):
        # 该节点的源目录路径
        self.source_path = source_path
        # 该节点生成的结果目录路径
        self.destination_path = destination_path
        # 子节点
        self.children = []
        # 节点类型：
        # 1. category 包含多个子目录 
        # 2. article 包含一个 index.md 文件和 images 目录
        # 3. leaf index.md 或者 images 目录
        self.node_type = node_type
        # 描述分类或者文章的元信息（比如：文章的标题，简介和日期）
        self.metadata = None
        
        Node.cache_map[source_path] = self

    def __str__(self):
        return f'path={self.source_path}'

```

### 遍历源目录

```python

def walk_dir(dir_path_str: str, destination_blog_dir_name: str) -> Node:
    """
    遍历目录，构造树结构
    :param dir_path_str: 存放博客 md 文件的目录的字符串
    :param destination_blog_dir_name: 生成博客目录的名称
    :return: 树结构的根节点
    """

    start = int(time.time() * 1000)
    q = deque()
    dir_path = Path(dir_path_str)
    q.append(dir_path)

    # 生成目录的根路径
    destination_root_dir = dir_path.parent.joinpath(destination_blog_dir_name)
    logger.info(f'源路经: {dir_path}, 目标路径: {destination_root_dir}')

    root = None

    # 层次遍历
    while q:
        item = q.popleft()
        if Path.is_dir(item):
            [q.append(e) for e in item.iterdir()]

        # node 类型判定
        node_type = 'leaf'
        if Path.is_dir(item):
            node_type = 'category'
            # 如果目录包含 index.md 则是文章目录节点
            for e in item.iterdir():
                if e.name == 'index.md':
                    node_type = 'article'
                    break

        if not root:
            root = Node(item, destination_root_dir, node_type)
        else:
            cur_node = Node.cache_map[item.parent]
            # 计算相对路径
            relative_path = item.relative_to(dir_path)
            # 构造目标路径
            destination_path = destination_root_dir / relative_path
            if destination_path.name == 'index.md':
                destination_path = destination_path.parent / Path('index.html')
            n = Node(item, destination_path, node_type)
            cur_node.children.append(n)
    end = int(time.time() * 1000)
    logger.info(f'构造树耗时: {end - start} ms')

    return root

```

### 生成目标目录

```python

def gen_blog_dir(root: Node):
    """
    根据目录树构造博客目录
    :param root: 树结构根节点
    :return:
    """

    start = int(time.time() * 1000)

    q = deque()
    q.append(root)

    # 清理之前生成的 root destination
    if Path.exists(root.destination_path):
        logger.info(f'存在目标目录: {root.destination_path}，进行删除')
        shutil.rmtree(root.destination_path)

    while q:
        node = q.popleft()
        [q.append(child) for child in node.children]

        # 对三种不同类型的节点分别进行处理

        if node.node_type == 'category' and node.source_path.name != 'images':
            Path.mkdir(node.destination_path, parents=True, exist_ok=True)
            category_index = node.destination_path / Path('index.html')
            categories = []
            for child in node.children:
                if child:
                    if child.node_type == 'article':
                        child.metadata = read_metadata(child.source_path / Path('index.md'))
                    relative_path = child.destination_path.name / Path('index.html')
                    categories.append({
                        'type': child.node_type,
                        'name': child.destination_path.name,
                        'href': relative_path,
                        'metadata': child.metadata,
                    })
            categories.sort(key=sort_categories)
            with open(category_index, mode='w', encoding='utf-8') as f:
                f.write(gen_category_index(categories, node.source_path.name))

        if node.node_type == 'article':
            Path.mkdir(node.destination_path, parents=True, exist_ok=True)

        if node.node_type == 'leaf':
            Path.mkdir(node.destination_path.parent, parents=True, exist_ok=True)
            if node.source_path.name == 'index.md':
                with open(node.destination_path, mode='w', encoding='utf-8') as f:
                    f.write(gen_article_index(node.source_path, node.source_path.parent.name))
            else:
                shutil.copy(node.source_path, node.destination_path)

    end = int(time.time() * 1000)
    logger.info(f'生成目标目录耗时: {end - start} ms')

```

### 其他辅助函数

```python

def md_to_html(md_file_path: Path) -> str:
    """
    markdown -> html
    :param md_file_path: markdown 文件的路径对象
    :return: html str
    """

    import markdown

    def remove_metadata(content: str) -> str:
        """
        删除文章开头的 YAML 元信息
        :param content: markdown 内容
        """
        lines = content.splitlines()
        if lines and lines[0] == '---':
            for i in range(1, len(lines)):
                if lines[i] == '---':
                    return '\n'.join(lines[i+1:])
        return md_content

    with open(md_file_path, mode='r', encoding='utf-8') as md_file:
        md_content = md_file.read()
        md_content = remove_metadata(md_content)
        return markdown.markdown(
            md_content,
            extensions=[
                'markdown.extensions.toc',
                'markdown.extensions.tables',
                'markdown.extensions.sane_lists',
                'markdown.extensions.fenced_code'
            ]
        )


def gen_article_index(md_file_path: Path, article_name):
    with open('./template/article.html', mode='r', encoding='utf-8') as f:
        bs1 = BeautifulSoup(f, "html.parser")
        bs2 = BeautifulSoup(md_to_html(md_file_path), "html.parser")
        bs1.find('article').append(bs2)
        bs1.find('title').string = f'文章 | {article_name}'
        return bs1.prettify()


def gen_category_index(categories: list, category_name) -> str:
    from jinja2 import Template

    with open('./template/category.html', mode='r', encoding='utf-8') as f:
        template_content = f.read()
        template = Template(template_content)
        html = template.render(categories=categories, category_name=category_name)
        return html


def sort_categories(item):
    """
    对 categories 排序，type = category 排在所有 type = article 前
    category 按照 name 字典顺序 a-z 排序
    article 按照 metadata 的 date 字段（格式：2024-02-03T14:44:42+08:00）降序排列。
    :param item:
    :return:
    """
    from datetime import datetime
    if item['type'] == 'category':
        # 分类优先，按 name 排序
        return 0, item['name'].lower()
    elif item['type'] == 'article':
        # 文章按日期降序排序，优先级次于 category
        # 将日期解析为 datetime 对象，若无日期则排在最后
        date = item['metadata'].get('date')
        parsed_date = datetime.fromisoformat(date) if date else datetime(year=1970, month=1, day=1)
        return 1, -parsed_date.timestamp()


def cp_css(dir_path_str: str):
    dir_path = Path(dir_path_str)
    destination_root_dir = dir_path.parent.joinpath('public').joinpath('css')
    shutil.copytree('./css', str(destination_root_dir.absolute()))


def read_metadata(md_file_path):
    import re
    with open(md_file_path, 'r', encoding='utf-8') as file:
        content = file.read()

    # 正则提取元数据
    match = re.match(r'^---\n([\s\S]*?)\n---\n', content)
    if match:
        metadata = match.group(1)
        return parse_metadata(metadata)
    return {}


def parse_metadata(metadata):
    """
    将元数据解析为字典
    title, date, summary
    """
    meta_dict = {}
    for line in metadata.split('\n'):
        if ':' in line:
            key, value = map(str.strip, line.split(':', 1))
            meta_dict[key] = value
    return meta_dict

```

---

## 样式

article.css

```css
* {
    margin: 0;
    padding: 0;
    box-sizing: border-box;
}

body {
    background-color: wheat;
}

article {
    margin: 0 auto;
    width: 60%;
    background-color: beige;
    padding-left: 20px;
    padding-right: 20px;
}

article h1 {
    font-size: 3em;
    text-align: center;
    margin-top: 30px;
    margin-bottom: 30px;
}

article h2 {
    font-size: 2.2em;
    margin-top: 30px;
    margin-bottom: 20px;
}

article h3, article h4, article h5, article h6 {
    font-size: 1.8em;
    margin-top: 10px;
    margin-bottom: 10px;
}

article p {
    font-size: 1.2em;
    line-height: 2.5em;
    margin-bottom: 20px;
}

article img {
    max-width: 100%; /* 限制图片宽度为容器宽度 */
    height: auto;    /* 保持图片比例 */
}

code {
    background-color: #f4f4f4; /* 设置浅灰色背景 */
    border: 1px solid #ddd;   /* 添加边框 */
    border-radius: 4px;       /* 圆角边框 */
    padding: 2px 4px;         /* 内边距 */
    font-family: monospace;   /* 使用等宽字体 */
    font-size: 0.95em;        /* 稍微缩小字体 */
    color: #c7254e;           /* 设置代码颜色 */
}

blockquote {
    margin: 1em 0;                 /* 设置上下间距 */
    padding: 0.5em 1em;            /* 内边距 */
    border-left: 4px solid #0074d9; /* 左侧蓝色边框 */
    background-color: #f9f9f9;     /* 浅灰背景 */
    color: #555;                   /* 设置文本颜色 */
    font-style: italic;            /* 倾斜字体 */
}

table {
    width: 100%;                  /* 表格宽度为100% */
    border-collapse: collapse;    /* 合并边框 */
    margin: 1em 0;                /* 设置表格的外边距 */
}

table th, table td {
    border: 1px solid #ddd;       /* 添加边框 */
    padding: 8px;                 /* 单元格内边距 */
    text-align: left;             /* 文本左对齐 */
}

table th {
    background-color: #f4f4f4;    /* 表头背景色 */
    color: #333;                  /* 表头文字颜色 */
    font-weight: bold;            /* 加粗表头文字 */
}

table tr:nth-child(even) {
    background-color: #f9f9f9;    /* 奇偶行背景交替 */
}

table tr:hover {
    background-color: #f1f1f1;    /* 鼠标悬停高亮 */
}

ul, ol {
    margin: 1em 0;               /* 上下外边距 */
    padding-left: 2em;           /* 左内边距（缩进） */
    line-height: 1.6;            /* 设置行高 */
}

ul {
    list-style-type: disc;       /* 使用圆点作为项目符号 */
}

ol {
    list-style-type: decimal;    /* 使用数字作为编号 */
}

ul li, ol li {
    margin-bottom: 0.5em;        /* 列表项的下间距 */
}

ul li::marker {
    color: #0074d9;             /* 修改圆点颜色 */
}

ol li::marker {
    color: #e74c3c;             /* 修改数字颜色 */
}

li {
    font-size: 1rem;             /* 设置字体大小 */
    color: #333;                 /* 设置文字颜色 */
}

li:hover {
    color: #0074d9;              /* 鼠标悬停时改变文字颜色 */
}

hr {
    margin-top: 40px;
    margin-bottom: 40px;
}

nav {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 10px 20px;
  background-color: #333;
  color: #fff;
}

nav .brand {
  font-size: 20px;
  font-weight: bold;
  text-decoration: none;
  color: #4caf50;
}

nav .nav-links a {
  margin-left: 15px;
  text-decoration: none;
  color: #fff;
  transition: color 0.3s;
}

nav .nav-links a:hover {
  color: #4caf50;
}

.site-footer {
    color: #fff;
    display: flex;
    justify-content: center;
    align-items: center;
    margin-bottom: 10px;
}

```


category.css

```css

* {
    margin: 0;
    padding: 0;
    box-sizing: border-box;
}

body {
    background-color: #202020;

    min-height: 100vh;
    display: flex;
    flex-direction: column;

}

h1 {
    text-align: center;
    color: moccasin;
    margin-top: 3%;
    margin-bottom: 3%;
}

h2 {
    text-align: center;
    color: rgb(121, 149, 146);
    margin-top: 3%;
    margin-bottom: 3%;
}

hr {
    border: 0;
    border-top: 1px dashed #a2a9b6;
    margin-top: 3%;
    margin-bottom: 3%;
}

.site-footer {
    color: #fff;
    display: flex;
    justify-content: center;
    align-items: center;
    margin-bottom: 10px;
}

.category-container {
    display: flex;
    justify-content: center;
    align-items: center;
    flex-direction: column;
}

.category {
    text-align: center;
    margin-top: 25px;
    margin-bottom: 25px;
    width: clamp(250px, 40%, 800px);
    border: 2px solid #ffe9a5;
}

.article {
    text-align: center;
    margin-top: 25px;
    margin-bottom: 25px;
    width: clamp(250px, 40%, 800px);
    border: 2px solid #c1f2ef;
}

.item-link {
    margin-top: 20px;
    margin-bottom: 10px;
}

a {
    font-size: 1em;
    text-decoration: none;
    color: #8b8ea3;
}

nav {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 10px 20px;
  background-color: #333;
  color: #fff;
}

nav .brand {
  font-size: 20px;
  font-weight: bold;
  text-decoration: none;
  color: #4caf50;
}

nav .nav-links a {
  margin-left: 15px;
  text-decoration: none;
  color: #fff;
  transition: color 0.3s;
}

nav .nav-links a:hover {
  color: #4caf50;
}

.article-metadata {
  color: #00c6e9;
}

```

---

## 总结

编写个人网站的过程完全是自由的（除了不能发布法律不允许的范围的信息之外，境外服务器另说），制造工具的过程本身，更能体会到这个工具为了解决的目标问题的含义。


