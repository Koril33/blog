---
title: "Python压缩JPEG图片"
date: 2023-05-20T10:41:51+08:00
tags: []
featured_image: "images/background.jpg"
summary: "使用Pillow压缩JPEG图片"
toc: true
---

## 前言

JPEG 格式图片的存储和压缩原理是一个非常复杂的话题，本文仅从日常脚本的使用来描述通过 Pillow 库压缩图片的方法。

---

## 最简单的 save

Pillow 库中的 Image.save 方法，可以指定保存的 quality，从而实现 JPEG 图片的压缩：

```python
from PIL import Image


def compress_image(file_path: str, quality: int):
    img = Image.open(file_path)
    img.save('compressed_img.jpg', "JPEG", optimize=True, quality=quality)
    return
```

过程很简单，打开一个图片，然后按照指定的 quality 保存图片。

quality 的范围是限制的，范围从1 到 100，但最好不要超过 95，不然体积会变得很大。

参考这篇文章：https://jdhao.github.io/2019/07/20/pil_jpeg_image_quality/

另外官方文档，也对 quality 做出了一定的解释：

https://pillow.readthedocs.io/en/stable/handbook/image-file-formats.html

> **quality**
>
> Integer, 0-100, Defaults to 80. For lossy, 0 gives the smallest size and 100 the largest. For lossless, this parameter is the amount of effort put into the compression: 0 is the fastest, but gives larger files compared to the slowest, but best, 100.

具体 quality 应该取多少，请根据实际情况判断。

---

## 改变图像分辨率

除了 quality 之外，假设一张图片的原始分辨率是 1920 × 1080，改变它的分辨率为 960 × 540，同样可以起到减小图片占用空间的效果：

```python
from PIL import Image


def resize_image(file_path: str, ratio: float = None, width: int = None, height: int = None):
    img = Image.open(file_path)
    origin_width, origin_height = img.size[0], img.size[1]
    if ratio and ratio < 1.0:
        img = img.resize(
            (
                int(origin_width * ratio),
                int(origin_height * ratio)
            )
        )
    elif width and height:
        img = img.resize(
            (width, height)
        )
    img.save('compressed_img.jpg')
    return
```

这里有三个参数，ratio、width、height，ratio 优先级高，如果指定了 ratio，则按照原来的分辨率按照比例缩小图片，如果未指定 ratio，而指定了 width 和 height，则直接 resize 成新的尺寸。

---

## 参考

1. https://www.thepythoncode.com/article/compress-images-in-python
2. https://pillow.readthedocs.io/en/stable/index.html
