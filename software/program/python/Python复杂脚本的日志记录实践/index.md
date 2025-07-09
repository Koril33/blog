---
title: "Python复杂脚本的日志记录实践"
date: 2025-07-09T16:00:00+08:00
summary: "Python 在复杂场景下，日志的最佳实践"
---

## 目录

[TOC]

---

## 前言

与那些一次性的简单脚本不同，复杂场景下的脚本，比如一些定时任务，或者系统运维脚本的运行时间较长，代码较为复杂，这时候简单的 basicConfig 可能就够用了。

Python 的 logging 提供了更加灵活的配置方案，使用字典配置的形式，并且支持自定义的 handler。

---

## 需求

在这些复杂，长时间运行的脚本中，日志系统大体上需要以下一些功能：

- 控制台和文件都要记录日志信息
- 日志文件需要能够自动轮转（按照文件大小或者间隔时间）
- 支持告警机制，可以在发生 error 级别的日志时，通过邮件或者其他方式告知运维人员

---

## 实现

日志配置 log_config.py 文件如下:

```python
import logging.handlers
import logging.config
import smtplib
from concurrent.futures import ThreadPoolExecutor
from email.header import Header
from email.mime.text import MIMEText
from email.utils import formatdate
from pathlib import Path
import requests

log_dir = Path('./logs')
log_dir.mkdir(parents=True, exist_ok=True)
app_log_file = log_dir / 'app.log'

app_name = 'project-test-app'
app_cn_name = '测试项目'

handler_executor = ThreadPoolExecutor(max_workers=10)


class SSLSMTPHandler(logging.handlers.SMTPHandler):

    def smtp_task(self, record):
        try:
            port = self.mailport
            if not port:
                port = smtplib.SMTP_SSL_PORT
            smtp = smtplib.SMTP_SSL(self.mailhost, port, timeout=self.timeout)
            message = self.format(record)

            # 构建MIME邮件，指定编码为UTF-8
            msg = MIMEText(message, 'plain', 'utf-8')
            msg['From'] = self.fromaddr
            msg['To'] = ','.join(self.toaddrs)
            msg['Subject'] = Header(self.subject, 'utf-8')
            msg['Date'] = formatdate()

            if self.username and self.password:
                smtp.login(self.username, self.password)
            smtp.sendmail(self.fromaddr, self.toaddrs, msg.as_string())
            smtp.quit()
        except Exception as e:
            app_logger.exception(f'smtp task failed: {e}')
            self.handleError(record)

    def emit(self, record):
        handler_executor.submit(self.smtp_task, record)


class NtfyHandler(logging.Handler):
    def __init__(self, ntfy_url, username, password, timeout=5):
        logging.Handler.__init__(self)
        self.ntfy_url = ntfy_url
        self.username = username
        self.password = password
        self.timeout = timeout

    def ntfy_task(self, record):
        try:
            response = requests.post(
                url=self.ntfy_url,
                auth=(self.username, self.password),
                data=self.format(record),
                timeout=self.timeout,
            )
            response.raise_for_status()
        except Exception as e:
            app_logger.exception(f'ntfy task failed: {e}')
            self.handleError(record)

    def emit(self, record):
        handler_executor.submit(self.ntfy_task, record)


my_config = {
    'version': 1,
    # 格式化器
    'formatters': {
        'simple_formatter': {
            'format': '%(asctime)s - %(name)s - %(filename)s - %(funcName)s - %(levelname)s :: %(message)s'
        },
        'ntfy_formatter': {
            'format': f'应用 <{app_name}|{app_cn_name}> 异常告警 - %(asctime)s - %(levelname)s :: %(message)s'
        }
    },
    # 处理器
    'handlers': {
        'console_handler': {
            'class': 'logging.StreamHandler',
            'formatter': 'simple_formatter',
            'level': logging.DEBUG,
            'stream': 'ext://sys.stdout'
        },
        'file_handler': {
            # 'class': 'logging.handlers.TimedRotatingFileHandler',
            'class': 'concurrent_log_handler.ConcurrentTimedRotatingFileHandler',
            'formatter': 'simple_formatter',
            'level': logging.DEBUG,
            'filename': app_log_file,
            'when': 'M',
            'backupCount': 3,
        },
        'ntfy_handler': {
            'class': 'log_config.NtfyHandler',
            'formatter': 'ntfy_formatter',
            'level': logging.WARNING,
            'ntfy_url': 'http://ntfy.example.com/app_notify',
            'username': 'user',
            'password': 'password',
        },
        'email_handler': {
            'class': 'log_config.SSLSMTPHandler',
            'formatter': 'simple_formatter',
            'level': logging.ERROR,
            'mailhost': ('smtp.163.com', 465),
            'fromaddr': 'example@163.com',
            'toaddrs': ['example@163.com'],
            'subject': f'应用 <{app_name}|{app_cn_name}> 异常告警',
            'credentials': ('example@163.com', 'your_email_password'),
            'timeout': 3,
            'secure': (),
        },
    },
    # 记录器
    'loggers': {
        'app': {
            'handlers': ['console_handler', 'file_handler', 'ntfy_handler', 'email_handler'],
            'level': logging.DEBUG,
            'propagate': False,
        }
    },
}

logging.config.dictConfig(my_config)
app_logger = logging.getLogger('app')
```

应用代码 main.py:

```python
from log_config import app_logger


def task(num):
    """
    执行一个简单的累加任务并在最后除以一个随机数字
    """
    import random
    app_logger.debug(f'task param <num> is {num}')

    result = 0
    for i in range(1, num+1):
        result += i
        app_logger.debug(f'task loop {i}, current result is {result}')

    # 随机制造一个除零异常，演示异常情况日志输出
    random_num = random.choice([0, 1, 2])
    app_logger.debug(f'task random_num is {random_num}')

    return result / random_num


def main():

    app_logger.info('main task start!')

    task_res = None
    try:
        task_res = task(4)
    except Exception as e:
        app_logger.exception(f'main task occur: {e}')

    if task_res:
        app_logger.info(f'main task got final result is {task_res}')
    else:
        app_logger.warning(f'task result is None.')


    app_logger.info('main task end...')


if __name__ == '__main__':
    main()
```


这里实现有几个点需要注意:

1. 使用 ThreadPoolExecutor 是因为 SMTP 或者 HTTP 之类的 handler 会有很明显的 IO 阻塞，需要放到线程池执行。
2. 使用第三方库的 ConcurrentTimedRotatingFileHandler 而不是标准库的 TimedRotatingFileHandler 是因为多进程下日志轮转会有并发问题。
3. 日志 handler 在发送日志信息的过程中也可能会碰到异常（尤其是依赖网络的 handler，比如：HTTP Handler），在 try-except 需要记录下来。

