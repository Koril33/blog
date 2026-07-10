---
title: "SpringBoot埋点"
date: 2026-07-01T16:36:41
summary: "在 Spring Boot 应用中实现自定义的埋点"
---

## 前言

springboot 通过引入 actuator 和 micrometer 可以实现对 Prometheus 暴露许多官方提供的 metrics（参考：[springboot接入prometheus](https://blog.djhx.site/software/prometheus/springboot%E6%8E%A5%E5%85%A5prometheus/index.html)）。

但是对于更加自定义的指标，则需要我们手动编写代码，这个过程就是“埋点”。

---

## 依赖和配置

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>

<dependency>
    <groupId>io.micrometer</groupId>
    <artifactId>micrometer-registry-prometheus</artifactId>
</dependency>

<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
</dependency>
```

```
management.endpoints.web.exposure.include=*
management.metrics.tags.application=${spring.application.name}
```
---

## metrics 类型

micrometer 支持的 metrics 类型如下：

- Timer
- Counter
- Gauge
- DistributionSummary
- LongTaskTimer
- FunctionCounter
- FunctionTimer
- TimeGauge

Prometheus 支持的 metrics 类型如下：

- Counter
- Gauge
- Histogram
- Summary

micrometer 和 prometheus 的类型没有一一对应，所以 micrometer 输出的指标会转换成 prometheus 的多个时间序列。

### Timer

用于统计一批已经完成的操作耗时，例如 HTTP 请求、SQL 查询、方法调用。



### Counter

用于统计只增不减的事件累计次数，例如：

- 请求数
- 登录失败数
- 订单创建数
- 消息消费数

只能增加正数，应用重启后可以归零。

如果已经使用 Timer 或 DistributionSummary，通常不需要再额外建 Counter，因为它们本身就包含 count。

### Gauge

表示某个对象的当前瞬时值，可以增加也可以减少，例如：

- 当前队列长度
- 当前线程数
- 当前连接数
- 集合大小
- 当前内存使用量

Gauge 只在采集时读取当前值，不保留两次采集之间发生过的变化。

### DistributionSummary

- 用于统计非时间类型数值的分布，例如：
- HTTP 响应体大小
- 订单金额
- 批处理记录数量
- 文件大小
- 压缩率

### LongTaskTimer

用于监控当前仍在运行的长任务，例如：

- 定时数据同步
- 大文件导入
- 数据库迁移
- 长时间批处理
- 模型训练任务

它通常提供：

- 当前活跃任务数
- 当前所有活跃任务的累计运行时间
- 当前最长任务的运行时间

和普通 Timer 的区别：

Timer：任务完成后才记录耗时

LongTaskTimer：任务还没完成时就能看到已经运行多久

因此可以在长任务超时但尚未结束时及时告警。

### FunctionCounter

也是累计计数器，但它不是通过 counter.increment(); 主动更新，而是通过一个函数读取现有对象中的累计值，例如：

cache.stats().evictionCount()

适合第三方对象已经维护了累计计数，而你只需要将它暴露为指标的情况。

被观察的函数值必须单调不减；Micrometer 不会替你检查这一点。

### FunctionTimer

通过两个函数读取已有对象中的：

累计执行次数
累计执行时间

例如某个缓存组件已经维护：

getOperationCount
totalGetLatency

此时可以用 FunctionTimer 直接适配，而不需要在每次调用时执行 timer.record()。
两个函数都应该是单调递增的累计值。后端可以据此计算：

吞吐量 = count 的增长速率
平均耗时 = totalTime 增长速率 / count 增长速率

### TimeGauge

一种带有时间单位信息的 Gauge，表示当前时间值，而不是一批事件的耗时分布，例如：

当前任务等待时间
当前连接空闲时间
缓存条目剩余 TTL
当前延迟估计值

Micrometer 会根据后端自动转换时间单位。例如输入 4000 ms，导出给 Prometheus 时会转换成：
my_gauge_seconds 4.0

它与 Timer 的区别：
TimeGauge：一个可以上下变化的当前时间值
Timer：多次已完成操作的耗时统计


