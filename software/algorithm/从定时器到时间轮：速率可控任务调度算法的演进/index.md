---
title: "从定时器到时间轮：速率可控任务调度算法的演进"
date: 2026-01-18T10:44:43
summary: "简单介绍各种定时器模式的算法"
---

## 前言

在工程中，定时器是一个非常常见的实际应用，从简单的定时备份日志，到复杂的速率可控的分布式爬虫系统，定时器的概念都作为其中的核心角色。

本文从最简单粗暴的 sleep 定时器到精确复杂的时间轮，逐个介绍各个算法的示例应用。

---

## sleep 定时器

对于单个 task 的定时器应用，sleep 是最简单的方法：

```python
import time

def task():
    print("run task")

while True:
    task()
    time.sleep(1) 
```

对于临时性的脚本，执行单个任务，这是最简单的方式，涉及到多个任务就需要使用多线程，每个线程控制一个任务的执行。

```python

import time
from concurrent.futures import ThreadPoolExecutor
from datetime import datetime


def task(task_id, delay):
    while True:
        print(f'{datetime.now()}: start task-{task_id}...')
        time.sleep(delay)
        print(f'{datetime.now()}: task-{task_id} done...')


def main():

    with ThreadPoolExecutor(max_workers=10) as executor:
        executor.submit(task, 1, 2)
        executor.submit(task, 2, 3)

if __name__ == '__main__':
    main()

```

打印结果：

```
2026-01-18 11:33:25.167645: start task-1...
2026-01-18 11:33:25.167840: start task-2...
2026-01-18 11:33:27.167766: task-1 done...
2026-01-18 11:33:27.167810: start task-1...
2026-01-18 11:33:28.167939: task-2 done...
2026-01-18 11:33:28.167981: start task-2...
2026-01-18 11:33:29.167889: task-1 done...
2026-01-18 11:33:29.167932: start task-1...
2026-01-18 11:33:31.168026: task-1 done...
2026-01-18 11:33:31.168082: start task-1...
2026-01-18 11:33:31.168095: task-2 done...
2026-01-18 11:33:31.168107: start task-2...
2026-01-18 11:33:33.168173: task-1 done...
2026-01-18 11:33:33.168222: start task-1...
```

在以下情况满足时，可以考虑应用这种方式：

- 任务数量不多
- 临时性的脚本
- 对于时间精度要求不高

但是如果需要长时间运行，并且任务量一旦上来了，这种方式就会出现内存占用量大的问题，因为一个任务就需要一个线程进行死循环，这种使用多线程的方式是很浪费的。

---

## 多任务定时器（遍历轮询）

多任务下使用多线程太浪费内存了，可以把多个任务放在一个列表里，然后单线程轮询，每隔一段时间遍历所有 tasks，看哪些 task 需要执行任务。

```python

import concurrent
import time
from concurrent.futures import ThreadPoolExecutor
from datetime import datetime

tasks = [
    {
        'id': 1,
        'interval': 1,
        'last': None
    },
    {
        'id': 2,
        'interval': 2,
        'last': None
    }
]

def task(task_id):
    print(f'{datetime.now()}: start task-{task_id}...')
    time.sleep(3)
    print(f'{datetime.now()}: task-{task_id} done...')

executor = ThreadPoolExecutor(max_workers=10)

def main():
    while True:
        for t in tasks:
            now = time.time()
            task_id = t.get('id')
            last_exec_time = t.get('last')
            if not last_exec_time or (now - last_exec_time > t.get('interval')):
                t['last'] = now
                executor.submit(task, task_id)
        time.sleep(0.1)


if __name__ == '__main__':
    main()

```

打印结果：

```
2026-01-18 12:05:19.217040: start task-1...
2026-01-18 12:05:19.217228: start task-2...
2026-01-18 12:05:20.218447: start task-1...
2026-01-18 12:05:21.219559: start task-1...
2026-01-18 12:05:21.219772: start task-2...
2026-01-18 12:05:22.217222: task-1 done...
2026-01-18 12:05:22.217392: task-2 done...
2026-01-18 12:05:22.220780: start task-1...
2026-01-18 12:05:23.218568: task-1 done...
2026-01-18 12:05:23.221702: start task-1...
2026-01-18 12:05:23.221806: start task-2...
2026-01-18 12:05:24.219668: task-1 done...
2026-01-18 12:05:24.219845: task-2 done...
```
通过当前时间和上一次执行时间之差和速率相关字段（interval参数）进行对比，可以判断是否需要执行任务，把需要执行的任务丢到线程池中执行，任务执行结束后就不会一直占用线程，比之前的线程内死循环要高效的多。


---

## 优先队列

使用轮询任务列表的方式依然有很大的优化空间，如果任务列表很大的话，这种轮询就会变得非常不精确，假设有 10000 个任务，轮询到第 199 个任务的时候，很可能位于前面的第 n 个任务，应该要触发了，但是由于还需要轮询完剩下 （10000 - 199 + n - 1）个任务，才能触发这个第 n 个任务。

而且轮询的 time.sleep 是固定的时间，导致误差被进一步放大。

为了解决这个精度的问题，可以采用优先队列的方式：

```python

import heapq
import time
from concurrent.futures import ThreadPoolExecutor
from datetime import datetime

tasks = []

def task(task_id):
    print(f'{datetime.now()}: start task-{task_id}...')
    time.sleep(3)
    print(f'{datetime.now()}: task-{task_id} done...')

executor = ThreadPoolExecutor(max_workers=10)


def add_task(task_id, interval):
    # next_time 作为 heapq 里元素的优先级
    heapq.heappush(tasks, (time.time(), task_id, interval))


def main():
    add_task(1, 1)
    add_task(2, 2)

    while True:
        # 查看优先队列的第一个元素，该元素的 next_time 最小，即：最接近现在的时间
        next_time, task_id, interval = tasks[0]
        now = time.time()
        if now >= next_time:
            heapq.heappop(tasks)
            executor.submit(task, task_id)
            heapq.heappush(tasks, (now + interval, task_id, interval))
        else:
            # 精确的睡眠，而不是一个固定值
            time.sleep(next_time - now)


if __name__ == '__main__':
    main()

```

使用“下一次执行时间”来作为 heapq 的优先级，这样 heapq 会维护一个高效的小顶堆，每次取出优先队列第一个元素就是最接近当前时间的任务。

这种方式很高效，不需要轮询全部任务，并且睡眠时间也不再是一个固定值，而是通过当前时间和下次执行任务时间的差值来确定。

---

## 令牌桶


---

## 时间轮
