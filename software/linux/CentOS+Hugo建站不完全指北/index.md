---
title: "CentOS+Hugo建站不完全指北"
date: 2021-05-08T16:33:58+08:00
summary: "CentOS7+Hugo+Nginx搭建网站"
featured_image: "images/background.jpg"
toc: true
---

## Hugo简介

Hugo是由Go语言实现的静态网站生成器。简单、易用、高效、易扩展、快速部署。

官网：<https://gohugo.io/>

---

## Hugo安装

官网安装文档：<https://gohugo.io/getting-started/installing>

### Windows

下载地址：<https://github.com/gohugoio/hugo/releases>

找到`hugo_0.83.1_Windows-64bit.zip`，下载之。

解压后，将`D:\hugo_0.83.1_Windows-64bit`添加到系统环境变量中。

打开cmd，键入命令`hugo version`检验是否添加成功：

```markdown
C:\Users\dingj>hugo version
hugo v0.83.1-5AFE0A57 windows/amd64 BuildDate=2021-05-02T14:38:05Z VendorInfo=gohugoio
```

---

## 在本地建站

官网快速入门文档：<https://gohugo.io/getting-started/quick-start/>

首先在本地某个目录下使用命令`hugo new site myblog`：

```markdown
PS E:\> hugo new site myblog
Congratulations! Your new Hugo site is created in E:\myblog.

Just a few more steps and you're ready to go:

1. Download a theme into the same-named folder.
   Choose a theme from https://themes.gohugo.io/ or
   create your own with the "hugo new theme <THEMENAME>" command.
2. Perhaps you want to add some content. You can add single files
   with "hugo new <SECTIONNAME>\<FILENAME>.<FORMAT>".
3. Start the built-in live server via "hugo server".

Visit https://gohugo.io/ for quickstart guide and full documentation.
```

在这个目录下会新建一个名叫 myblog 的文件夹，这个就是本地的网站目录了。

---

## 挑选主题

主题站点：<https://themes.gohugo.io/>

我挑选了一个简约干净的主题：<https://github.com/joway/hugo-theme-yinyang>

把这个主题克隆到 themes 目录下：

```markdown
PS E:\> cd .\myblog\
PS E:\myblog> git clone git@github.com:joway/hugo-theme-yinyang.git themes/yinyang
Cloning into 'themes/yinyang'...
remote: Enumerating objects: 773, done.
remote: Counting objects: 100% (91/91), done.
remote: Compressing objects: 100% (58/58), done. eceiving objects:   0% (1/773)
remote: Total 773 (delta 46), reused 67 (delta 27), pack-reused 682Receiving objects:
Receiving objects: 100% (773/773), 900.26 KiB | 749.00 KiB/s, done.
Resolving deltas: 100% (407/407), done.
```

---

## 更改配置文件

myblog 下的 config.toml 就是网站的配置文件。主题克隆后，要记得在配置文件中配置：`theme = "yinyang"`

下面是我的配置文件：

```toml
baseURL = "http://www.korilweb.cn/"
languageCode = "en-us"
title = "Koril's Blog"
# 设置主题
theme = "yinyang"
DefaultContentLanguage = "cn"

[author]
  name = "Koril"
  homepage = "http://www.korilweb.cn/"

[params]
headTitle = "Koril Web"
mainSections = ["posts"]
[[params.socials]]
name = "About Me"
link = "http://www.korilweb.cn/AboutMe"
[[params.socials]]
name = "Github"
link = "https://github.com/Koril33"
```

---

## 发布文章

键入命令：hugo new posts/my-first-post.md，就可以发表一篇名为 my-first-post 的文章，格式为markdown。

```markdown
PS E:\myblog> hugo new posts/my-first-post.md
E:\myblog\content\posts\my-first-post.md created
```

在 content\posts 目录下可以看到 my-first-post.md，打开进行编辑。

默认创建后，文档开头会有几个属性：

```markdown
title: "My First Post"
date: 2021-05-08T17:00:15+08:00
draft: true
```

**值得注意的是**，文档的 draft 属性如果是 true 的话，hugo默认不发布这篇文档，但可通过命令`--buildDrafts`来命令hugo发布“草稿”，如果写好一篇文章，记得把 draft 设置为 false。

关于草稿的详细内容可参考：<https://gohugo.io/getting-started/usage/#draft-future-and-expired-content>

---

## 预览

Hugo 有个很好的功能，在文档中编辑，可以实时显示在网页中，键入命令 `hugo server --buildDrafts`（或者`hugo server -D`）：

```markdown
PS E:\myblog> hugo server --buildDrafts
Start building sites …

                   | CN
-------------------+-----
  Pages            |  9
  Paginator pages  |  0
  Non-page files   |  0
  Static files     |  4
  Processed images |  0
  Aliases          |  0
  Sitemaps         |  1
  Cleaned          |  0

Built in 14 ms
Watching for changes in E:\myblog\{archetypes,content,data,layouts,static,themes}
Watching for config changes in E:\myblog\config.toml
Environment: "development"
Serving pages from memory
Running in Fast Render Mode. For full rebuilds on change: hugo server --disableFastRender
Web Server is available at http://localhost:1313/ (bind address 127.0.0.1)
Press Ctrl+C to stop
```

打开 <http://localhost:1313/> 就可以看到页面了。

---

## 设置  categories

在文档开头加入 categories 属性：

```markdown
title: "My First Post"
date: 2021-05-08T17:00:15+08:00
draft: true
categories: ["科技"]
```

---

## 打包成静态网站

写好了一篇文章后，就可以进行打包了，键入命令：`hugo`即可（如果要连草稿一起打包，使用`hugo -D`）:

```markdown
PS>hugo -D
Start building sites …

                   | CN
-------------------+-----
  Pages            | 11
  Paginator pages  |  0
  Non-page files   |  0
  Static files     |  4
  Processed images |  0
  Aliases          |  0
  Sitemaps         |  1
  Cleaned          |  0

Total in 212 ms
```

可以看到在 myblog 目录下多了一个 public 文件夹，这就是所有的静态页面。

---

## 部署至服务器

如果没有云服务器的同学，可以使用 GitHub Page + Hugo 的解决方案，就是把 public 文件夹托管到 GitHub 上。网上有很多教程，这里不再赘述。

我用的是腾讯云的服务器，使用 Nginx 来做路由配置。

在根目录下新建一个data目录，存放所有的网站文件：

```markdown
[root@VM-0-4-centos ~]# mkdir /data
```

然后把本地的 public 文件夹放到服务器上：

```markdown
scp -r .\public\ root@你的服务器IP地址:/data/
```

最后，配置一下 nginx：

```markdown
[root@VM-0-4-centos ~]# vim /usr/local/nginx/conf/nginx.conf
```

配置 server 中的 location 即可：

```nginx
location / {
    root     /data/public/;
    index    index.html index.htm;
}
location /AboutMe {
    alias     /data/site/;
    index    index.html index.htm;
}
```

第一个 location 指向了 public，也就是hugo打包好的静态页面，第二个 location 指向 site 文件夹，我在里面放了一个 index.html 来作 AboutMe 页面（上面的配置文件有配置AboutMe的URL是 http://www.korilweb.cn/AboutMe 这里与之相对应）。

> 注意 root 和 alias 有不同的作用，root 相当于是指定了根目录，会与 location 后的路径相连接，形成完整的路径。而 alias 则是会直接替换 location 后面的路径。
>
> 比如：
>
> ```nginx
> location /file/ {
>     root      /data/public/;
> }
> location /image/ {
>     alias     /data/site/;
> }
> ```
>
> 如果一个请求的URI是/file/index.html时，服务器返回的是：/data/public/file/index.html
>
> 如果一个请求的URI是/image/index.html时，服务器返回的是：/data/site/index.html
>
> 使用alias时，目录名后面一定要加"/"。

最后，启动 nginx：`nginx -c /usr/local/nginx/conf/nginx.conf `。

