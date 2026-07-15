---
title: "springboot接入prometheus"
date: 2026-06-27T21:10:58
summary: "Prometheus 收集 Spring Boot 的 metrics"
---

## 目录

[TOC]

---

## Micrometer&Actuator

Spring 有个项目叫 micrometer, 用来收集应用的各项指标，提供了对于 metrics 收集的统一抽象，最终的数据可以流入到 Prometheus、influxdb。

另一个项目叫 actuator, 用来将 micrometer 收集的指标数据暴露成 API，供外界访问。

micrometer 和 actuator 给开发人员提供了可观测性的能力，让运行的 Spring Boot 可以暴露出很多有用的指标和信息。

---

## 依赖和基本配置

maven：

```xml
<!-- Web -->
<dependency>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-starter-web</artifactId>
</dependency>

 <!-- Actuator -->
<dependency>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>

<!-- Prometheus Registry -->
<dependency>
  <groupId>io.micrometer</groupId>
  <artifactId>micrometer-registry-prometheus</artifactId>
</dependency>
```

Spring Boot 配置：

```properties
management.endpoints.web.exposure.include=*
management.metrics.tags.application=${spring.application.name}
```

为了方便学习，这里开启了所有的 actuator 端口，因为 actuator 很多 API 会暴露一些敏感信息，所以不配置下，默认仅暴露 `/actuator/health`。

启动 Spring Boot，确认 actuator 能访问以后，配置 Prometheus：

```yml
  - job_name: springboot-web

    metrics_path: /actuator/prometheus
    static_configs:
      - targets: ['localhost:8080']
```

配置好后，重新载入配置

```shell
sudo systemctl reload prometheus
```

---

## 参考

1. [https://docs.spring.io/spring-framework/reference/integration/observability.html](https://docs.spring.io/spring-framework/reference/integration/observability.html)
2. [https://docs.spring.io/spring-boot/reference/actuator/observability.html](https://docs.spring.io/spring-boot/reference/actuator/observability.html)
3. [https://docs.spring.io/spring-boot/reference/actuator/monitoring.html](https://docs.spring.io/spring-boot/reference/actuator/monitoring.html)
4. [https://docs.spring.io/spring-boot/reference/actuator/metrics.html#actuator.metrics.export.prometheus](https://docs.spring.io/spring-boot/reference/actuator/metrics.html#actuator.metrics.export.prometheus)
5. [https://docs.spring.io/spring-boot/reference/actuator/endpoints.html#actuator.endpoints.exposing](https://docs.spring.io/spring-boot/reference/actuator/endpoints.html#actuator.endpoints.exposing)
