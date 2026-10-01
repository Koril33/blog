---
draft: true
title: "cachetools的使用"
date: 2026-08-09T14:12:17
summary: "缓存工具的一些使用场景"
---

## 缓存的基本概念

缓存是为了在某些存在消耗性的资源使用或者类似场景下，为了节省延迟时间而采取的技术，缓存的本质是空间换时间的做法：通过把一些数据预先或者事后存储在程序能够访问更快的空间中，来达到性能提升的目的。

缓存的优点是减少计算、减少 IO、降低数据库或者计算节点的负载，提高响应速度，简单的说，只有一个优点就是性能好。

但缓存存在非常多的缺点（或者说是设计缓存时需要考虑的问题），空间占用（如果是缓存在内存中，则需格外注意），数据的有效性和一致性，缓存穿透/缓存击穿/缓存雪崩，数据载入缓存的时机（冷启动和懒加载）。

可以把缓存的数据结构看作是一种特殊的 map。普通的 map 主要解决了 key - value 的映射，缓存则在此基础上额外增加了一些策略算法，比如决定一个 value 可以存储多久，什么时候删除，容量满了以后如何执行删除等等。

### key

缓存的 key 是需要仔细考虑的，为了避免命名冲突，需要考虑好应用的命名空间，如果有多个缓存版本，则需要考虑版本号，为了缓存数据库查询结果而设计的缓存还需要考虑查询参数组合的各种可能性。

通用的 key 设计可以遵循以下格式：

```
[环境前缀]:[应用名/命名空间]:[版本号]:[业务模块]:[实体类型]:[标识符/参数摘要]
```

比如，一个账号相关的应用，为了缓存用户信息。
复杂的版本：

```
prod:account_service:v1:user:145
```

一些简单应用，也可以忽略环境或者版本号：

```
account_service:v1:user:145
```

在大型的分布式项目中设计的缓存系统，团队对于 key 的命名应该有自己的固定规范，主要的原则：

- 唯一性：一个键必须唯一对应一份缓存数据，避免覆盖。
- 可读性：人能快速看懂键的含义，方便调试和监控。
- 简洁性：在可读的前提下尽量短，节省内存（Redis 里键本身占空间）。
- 确定性：相同的逻辑数据，无论参数顺序如何，只要含义相同，就应生成同一个键。
- 防冲突：不同业务、不同环境、不同应用的缓存键要隔离开。

上面只有一个查询条件（user id），而对于包含查询条件的缓存则需要注意查询字段的顺序，对于缓存 key 而言，下面的两个 key 可能是不同的：

```
user:age=20:city=Shanghai

user:city=Shanghai:age=20
```

确保相同语义的多个查询字段，最终能根据某个排序规则构造出相同的 key。

### value

key 按照普遍的规范都是字符串类型，但是 value 则可能是各种各样形式的对象，value 需要考虑的问题如下：

1. 大小，缓存空间往往在内存里，所以要避免把列表或者大对象一整个缓存到空间中，特别是一旦没有限制缓存的数量，就会慢慢蚕食内存。
2. 序列化，在使用外部缓存空间（比如 Redis），需要考虑合适的序列化方案，如果是存在跨语言的调用，可以使用 JSON 或者 Protobuf 等方案。
3. 可变性，在使用内存缓存（cachetools），确保缓存的对象字段不可变，cachetools 缓存的是对象引用，后续代码如果意外的修改了对象字段值，缓存就“脏”了，至少也要在取用的时候，使用 deepcopy 确保不要修改缓存的 value。


### 命中率

缓存有个非常重要的指标就是缓存的命中率：Hit Ratio = Hit / (Hit + Miss)。

Hit 就是命中数量，Miss 就是未命中数量，典型的请求流程是：

```

请求数据->缓存命中->直接返回

请求数据->缓存未命中->查询DB->缓存结果->返回

```

为了尽可能的减少 DB 的 IO 次数，缓存的命中率越高越好。为了确保缓存空间不要太大，我们会限制缓存数量，为了保证数据的新鲜度，我们会限制数据的有效期，而缓存数量和数据有效期都与命中率密切相关，缓存系统的设计就是在这些方面进行取舍。

### 缓存数据的载入时机

缓存的数据，什么时候，由谁来载入到缓存空间，也有很多的方式。



---

## 淘汰算法

缓存数据不可能一直存在于缓存空间里，因为数据会发生更新或者删除，应用要确保用户获取的数据的新鲜度，所以就产生了一系列的缓存数据的淘汰算法。

### FIFO

队列模式（First In First Out），假设缓存空间长度是三，目前有三个缓存数据，第四个缓存数据载入后，最先进入的缓存数据将被剔除。

```
A B C
```
D 载入后：

```
B C D
```

这是最简单的一种淘汰算法，队列的模式，先进先出，不关心数据的访问量和访问频率，只看数据的载入时间。

### LFU

最不经常使用（Least Frequently Used），LFU 会淘汰掉历史访问次数最少的元素：

```
A: 10 次
B: 5 次
C: 21 次

```

B 由于访问次数最少，会被该策略淘汰。


### LRU

最近最少使用（Least Recently Used），LRU 会按照访问时间排序，淘汰掉最后一个元素：

```

A: 前 5 秒被访问
B: 前一个小时被访问
C: 前 3 分钟被访问

```

B 由于被访问时的时间最久远，会被该策略淘汰掉。


### RR

随机删除（Random Replacement），随机选择一个元素删除。

### TTL

生存过期时间（Time To Live），使用该策略，每个元素会有一个存活时间，超过生存时间的缓存项将无法访问，并最终被移除。

### TLRU

带自定义过期时间的 LRU（Time aware Least Recently Used ），和 TTL 类似，但是 TTL 每个元素是固定的过期时间，实际生产中可能会碰到不同元素有不同的过期时间，这个时候就可以用 TLRU，该策略为每个缓存项关联一个生存时间值。超过生存时间的缓存项将无法访问，并最终被移除。如果没有过期的缓存项需要移除，则会优先丢弃最近最少使用的缓存项以腾出空间。

可以看作是可以为每个元素制定特定过期时间的 TTL + LRU 的合体。

---

## cachetools

cachetools 是 Python 生态中最常用的缓存库，它提供基于不同缓存算法的多个缓存类，以及用于缓存函数和方法调用的装饰器。

### FIFOCache

FIFOCache 按照进入缓存空间的时间，仅缓存最近进入的 N 条数据，在实际业务中使用的比较少：

```python

from cachetools import FIFOCache

c = FIFOCache(maxsize=3)

def main():
    c['a'] = 1
    c['b'] = 2
    c['c'] = 3
    print(c)
    print(f'cache len is {len(c)}')

    c['d'] = 4
    print(c)

    item = c.popitem()
    print(f'pop item is: {item}, cache: {c}')
    
    c.clear()
    print(c)
    print(f'cache len is {len(c)}')

if __name__ == "__main__":
    main()

```

FIFO 不关心热点数据，只是一个先进先出的队列结构，可以用在一些无关访问频率次数的场景里：

- 异步后台任务每隔一段时间的运算或者执行，会产生一个临时的中间数据，想要对近 N 分钟的中间数据做缓存。

### LFUCache

LFUCache 在缓存满了以后，优先淘汰“历史访问次数最少”的数据。

```python

from cachetools import LFUCache

c = LFUCache(maxsize=3)

def main():
    c['a'] = 1
    c['b'] = 2
    c['c'] = 3
    print(c)
    print(f'cache len is {len(c)}')

    _ = c['c']
    _ = c['a']

    c['d'] = 4
    print(c)

    item = c.popitem()
    print(f'pop item is: {item}, cache: {c}')
    
    c.clear()
    print(c)
    print(f'cache len is {len(c)}')

if __name__ == "__main__":
    main()

```

这里 key=a 和 key=c 各被访问一次，key=b 由于没有被访问所以被淘汰。

LFU 适合的场景是：数据访问存在明显的“热点”，而且热点数据会持续被访问，比如商品详情缓存，有些商品特别热门，访问量远高于其他冷门产品，就很适合做 LFU 缓存。

LFU 碰到以下场景，就可能不太合适了：

- 历史数据热点，但是并非会被持续访问，比如去年的某个产品特别热门，今年几乎没有访问量，但因为历史访问量非常大，导致一直占据在缓存空间中。
- 数据没有明显热点，所有 key 历史访问次数较为平均，那么 LFU 的优势就无法体现了。

### LRUCache

LRUCache 会在缓存满了以后，淘汰“最近最久没有被使用”的数据，和 LFU 关注热点数据，但是 LRU 是从时间维度来体现的。

```python

from cachetools import LRUCache

c = LRUCache(maxsize=3)

def main():
    c['a'] = 1
    c['b'] = 2
    c['c'] = 3
    print(c)
    print(f'cache len is {len(c)}')

    _ = c['c']
    _ = c['a']
    _ = c['b']
    _ = c['c']

    c['d'] = 4
    print(c)

if __name__ == "__main__":
    main()

```

这里访问顺序是 c->a->b->c，c 是最近一次访问到的，所以属于“热点”数据，a 是距离现在最早被访问的，所以 a 会被淘汰。

LFU 能应用的场景，LRU 也可以应用，都是为了缓存热点数据而存在的，但 LFU 是从历史访问频次来体现数据的热点程度，LRU 更关心的是数据最近被访问的时间，LRU 更适合热点频繁变动的场景。

假设去年的某个产品被访问了 100 万次，对于 LFU 而言 ，该产品可能非常重要（历史访问次数远远大于其他产品）。

但是对于 LRU 而言，因为今年这个产品没有被访问过了（最后一次访问时间是在去年），那么该产品的重要性就远远低于一个刚刚被访问的冷问产品（历史访问次数可能非常少）。

LRU 不关心历史访问次数，所以比 LFU 更快地适应热点变化。

### RRCache

RRCache 比 LRUCache/LFUCache 简单多了，就是在缓存满了以后，随机选择一个已有的 Key 淘汰。

```python

from cachetools import RRCache

c = RRCache(maxsize=3)

def main():
    c['a'] = 1
    c['b'] = 2
    c['c'] = 3
    print(c)
    print(f'cache len is {len(c)}')

    c['d'] = 4
    print(c)

if __name__ == "__main__":
    main()

```

在缓存满了以后，每次执行都是随机一个 key 被替换掉，这种机制适合一些无法判断什么数据属于热点的场景，或者一些访问非常平均的场景。

### TTLCache

TTLCache 和以上的淘汰策略相比，它关心的是数据的生存时间，所以更准确的称呼应该是：过期策略。

```python

from cachetools import TTLCache
import time
c = TTLCache(maxsize=3, ttl=10)

def main():
    c['a'] = 1
    c['b'] = 2
    c['c'] = 3
    print(c)
    print(f'cache len is {len(c)}')

    time.sleep(5)
    c['d'] = 4
    print(c)
    
    time.sleep(5)
    print(c)

    time.sleep(5)
    print(c)

if __name__ == "__main__":
    main()

```

TTLCache 的 ttl 是针对每个元素的，上面的代码表示 TTLCache 最多容纳 3 个元素，每个元素的生存时间是 10 秒（从放入缓存的时间开始计时）。

TTL 没有热点的概念，不管某个产品数据最近是否被频繁访问，历史访问频次是否最多，一旦 TTL 到期了，就会被删除，这种特性适合一些拥有固定生命周期的缓存数据，比如验证码之类的。

TTLCache 缓存满了的情况下，如果没有过期的缓存项需要移除，则会优先丢弃最近最少使用的缓存项以腾出空间（LRU）。

### TLRUCache

TLRUCache 可以看成是 TTLCache 的一个更灵活版本，TTLCache 是所有缓存项统一 TTL时间，而在 TLRUCache 中，每个缓存项可以根据自己的情况决定什么时候过期。



### getsizeof


---

## 参考

1. https://magicliang.github.io/2026/05/14/%E7%BC%93%E5%AD%98%E7%B3%BB%E7%BB%9F%E8%AE%BE%E8%AE%A1%E5%85%A8%E6%99%AF/index.html
2. https://cachetools.readthedocs.io/en/stable/
