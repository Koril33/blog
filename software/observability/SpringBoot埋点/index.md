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

- Timer：
- Counter：
- Gauge：
- DistributionSummary：
- LongTaskTimer：
- FunctionCounter：
- FunctionTimer：
- TimeGauge：

Prometheus 支持的 metrics 类型如下：

- Counter
- Gauge
- Histogram
- Summary
