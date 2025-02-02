---
title: "Docker部署一个Gunicorn+Flask应用"
date: 2023-05-02T10:26:48+08:00
tags: []
featured_image: "images/background.jpg"
summary: "一个照片处理的小应用为例子，在容器中部署 Flask 应用"
toc: true
---

## 前言

Python 在做网络爬虫，自动化运维，大数据，人工智能，图像处理等领域时，拥有着丰富的资源和完善的社区，往往在一个大型应用中，我们希望 Python 能够提供这些服务，暴露 API 供其他服务端调用，本文以 Flask 这个 Python Web 框架举例子，结合 Pillow 图像处理库，编写一个简单的应用，并将其部署于容器中。

本文额外使用的库仅包含：Flask，Pillow，Gunicorn。

requirements.txt 如下：

```
blinker==1.6.2
click==8.1.3
colorama==0.4.6
Flask==2.3.2
gunicorn==20.1.0
importlib-metadata==6.6.0
itsdangerous==2.1.2
Jinja2==3.1.2
MarkupSafe==2.1.2
Pillow==9.5.0
Werkzeug==2.3.3
zipp==3.15.0
```

----

## 一个简单的图像服务

提到图像处理，Python 中一个最常用的库就是 Pillow，它提供了广泛的文件格式支持、强大的图像处理能力、主要包括图像储存、图像显示、格式转换以及基本的图像处理操作等。

```python
from flask import Flask, request, jsonify
from PIL import Image


app = Flask(__name__)


@app.route('/')
def hello_world():  # put application's code here
    return 'Hello World!'


@app.post('/img-info')
def img_info():
    """
    解析客户端发来的图片，并返回图片信息
    :return: 图片信息
    """
    img_file = request.files['file']
    img = Image.open(img_file)
    info = {
        "size": img.size,
        "mode": img.mode
    }

    return jsonify(info)


if __name__ == '__main__':
    app.run()
```

服务很简单，接受客户端的一张图片，返回 size 和 mode 信息的 Json 格式数据。

---

## Gunicorn

Flask 是一个 web 应用框架，它本身并不包括 Web Server，为了开发和测试的方便，Flask 内置了一个 Werkzeug wsgi server，但是这个 server 并不高效，仅仅用于开发环境， 如果是在生产环境下部署的话， 就需要用 Gunicorn 去替代掉这个内置的 Wsgi Server。

gunicorn_config.py 配置文件

```python
bind = "0.0.0.0:5000"
workers = 4
threads = 4
timeout = 120
```

---

## Dockerfile

```dockerfile
FROM python:3.8-slim
# 根目录下创建一个文件夹，存放代码文件
RUN mkdir /my-flask-app
# 将其作为工作目录
WORKDIR /my-flask-app
# 将 requirements.txt 拷贝到工作目录下
ADD requirements.txt /my-flask-app
# 安装依赖
RUN pip3 install -r requirements.txt
# 将源代码拷贝到工作目录下
ADD . /my-flask-app
# 暴露端口 5000
EXPOSE 5000
# 使用 Gunicorn 启动 Flask 项目
ENTRYPOINT ["gunicorn", "--config", "/my-flask-app/gunicorn_config.py", "app:app"]
```

---

## 构建镜像以及启动

```shell
# 构建镜像
docker build -t koril/my-flask-app .
# 启动
docker run --name flask-app -dp 5000:5000 koril/my-flask-app
```

---

## 参考

1. https://blog.entirely.digital/docker-gunicorn-and-flask/
2. https://medium.com/thedevproject/setup-flask-project-using-docker-and-gunicorn-4dcaaa829620
