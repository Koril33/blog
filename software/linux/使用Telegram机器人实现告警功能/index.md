---
title: "使用Telegram机器人实现告警功能"
date: 2026-03-12T10:24:49
summary: "Telegram Bot——方便简单的私人告警"
---

## 目录

[TOC]

---

## 前言

国内想要做机器人告警的功能，可以选择诸如钉钉、QQ、飞书、企业微信等等平台，但是对于个人来说，想要跑完完整的流程实在是有些麻烦，因为国内的审核非常严格（会越来越严格），大部分的工具都需要接入大公司（腾讯、阿里）的开发者平台，然后需要很多资质的认证（个人身份证、企业资质），这对于公司来说肯定没问题，合规合法就能搞，但是对于个人开发者就很麻烦了，没法做企业资质认证，那么只能走第三方平台来推送消息或者自建服务（比如：ntfy）。

本文提供一个新的选择——telegram bot，使用非常简单，也无需认证。

---

## BotFather

首先需要建立一个机器人，在 @BotFather 下按一下顺序建立好机器人：

```
1. /start
2. /newbot
3. 输入一个独一无二的机器人名字
4. /mybots
5. 保存好返回的 token
```

创建好机器人后，就需要获取个人的 chat id，这样机器人才能知道往哪里发消息：

```
1. 先给你的机器人发一条消息

2. 调用一下接口，从返回的 JSON 中获取 chat.id
GET https://api.telegram.org/bot{填写你刚刚获取的token}/getUpdates
```

获取到机器人的 token 以及个人的 chatId 就可以用机器人给自己发消息了：

```
POST application/json

https://api.telegram.org/bot{token}/sendMessage

{
    "chat_id": "{chat.id}",
    "text": "Hello, telegram bot!",
    "parse_mode": "HTML"
}

```

---

## python-telegram-bot

如果是长期使用或者更加复杂的场景可以使用[python-telegram-bot](https://pypi.org/project/python-telegram-bot/)。

有两种方式响应用户信息和连接 TG 服务器：轮询（polling）或者回调接口（webhook）。

### 轮询

轮询适用于没有公网访问地址的内网机器，本质就是按照一定时间间隔，不断轮询 TG 服务器，看看有没有新的用户消息或者需要发送给用户的信息。

下面是一个内存告警的轮询模式的示例：

```python
import psutil
from telegram import Update
from telegram.ext import ApplicationBuilder, ContextTypes, CommandHandler

# 配置你的信息
TOKEN = '你的bot的token'
CHAT_ID = 123  # 你的数字 Chat ID
THRESHOLD = 90.0  # 内存报警阈值（百分比）


async def status(update: Update, context: ContextTypes.DEFAULT_TYPE):
    cpu = psutil.cpu_percent()
    mem = psutil.virtual_memory().percent
    text = f"📊 <b>当前系统状态</b>\nCPU: {cpu}%\n内存: {mem}%"

    # 回复用户
    await update.message.reply_text(text, parse_mode='HTML')

async def check_memory_job(context: ContextTypes.DEFAULT_TYPE):
    """定时检查内存的任务"""
    memory = psutil.virtual_memory()
    usage = memory.percent

    print(f'当前使用率: {usage:.2f}%')

    # 如果超过阈值，发送告警
    if usage > THRESHOLD:
        msg = (
            f"⚠️ <b>服务器内存告警</b>\n"
            f"当前使用率: {usage}%\n"
            f"可用内存: {memory.available / (1024 ** 3):.2f} GB"
        )
        await context.bot.send_message(chat_id=CHAT_ID, text=msg, parse_mode='HTML')


def main():
    # 初始化 Application
    application = ApplicationBuilder().token(TOKEN).build()

    # 注册定时任务
    job_queue = application.job_queue
    job_queue.run_repeating(check_memory_job, interval=10, first=0)

    # 注册命令处理器
    application.add_handler(CommandHandler('status', status))

    print("监控服务已启动...")

    application.run_polling()


if __name__ == '__main__':
    main()
```

### 回调函数

回调函数的性能比轮询更好，节省资源，但缺点是需要配置了 https 的公网域名。

下面是 FastAPI 的示例代码，提供给 telegram 一个 webhook，也就是回调接口，一旦 bot 收到用户信息，telegram 就会触发该接口：

```python

import os
from datetime import datetime

from dotenv import load_dotenv
from fastapi import FastAPI, Request
from telegram import Update
from telegram.ext import ApplicationBuilder, CommandHandler, ContextTypes

load_dotenv()

TOKEN = os.getenv("BOT_TOKEN")
# 可能需要挂代理才能访问 telegram
# PROXY_URL = "你的代理"

app = FastAPI()
tg_app = (
    ApplicationBuilder()
    .token(TOKEN)
    # .proxy(PROXY_URL)
    # .get_updates_proxy(PROXY_URL)
    .build()
)


async def echo(update: Update, context: ContextTypes.DEFAULT_TYPE):
    user_text = " ".join(context.args) if context.args else ""
    await update.message.reply_text(f"Server Echo: {user_text}")


async def get_datetime(update: Update, context: ContextTypes.DEFAULT_TYPE):
    now = datetime.now().strftime("%Y-%m-%d %H:%M:%S")
    await update.message.reply_text(f"Server Datetime:\n{now}")


tg_app.add_handler(CommandHandler("echo", echo))
tg_app.add_handler(CommandHandler("datetime", get_datetime))


@app.post("/tg-webhook")
async def telegram_webhook(request: Request):

    data = await request.json()

    update = Update.de_json(data, tg_app.bot)

    await tg_app.process_update(update)

    return {"status": "ok"}


@app.on_event("startup")
async def startup():

    await tg_app.initialize()

    webhook_url = "https://你的域名/tg-webhook"

    await tg_app.bot.set_webhook(webhook_url)

    print("Webhook 已设置:", webhook_url)

```

这里的 webhook 需要准备 https 和公网能访问到的域名，启动：

```shell
uvicorn app:app --host 0.0.0.0 --port 8080
```

nginx 需要配置好反向代理：

```
server {
  listen 80;
  server_name 你的域名;

  return 301 https://$host$request_uri;
}


server {
  listen 443 ssl;
  server_name 你的域名;

  ssl_certificate /etc/nginx/ssl/你的域名_nginx/你的域名_bundle.crt;
  ssl_certificate_key /etc/nginx/ssl/你的域名_nginx/你的域名.key;

  location /tg-webhook {
    proxy_pass http://127.0.0.1:8080/tg-webhook;
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;
  }
}
```

---

## 参考

1. https://www.cnblogs.com/xhzhang/p/19371452
2. https://pypi.org/project/python-telegram-bot/