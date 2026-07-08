# Spring Cloud 生态详解

> Spring Cloud 是国内微服务架构的事实标准。面试中不会 Spring Cloud 等于不会微服务。本篇覆盖注册中心、负载均衡、服务调用、熔断、网关、配置中心、链路追踪全链路。

---

## 目录

- [1. 整体架构概览](#1-整体架构概览)
- [2. 服务注册与发现](#2-服务注册与发现)
- [3. 负载均衡](#3-负载均衡)
- [4. 服务调用](#4-服务调用)
- [5. 熔断与降级](#5-熔断与降级)
- [6. 网关](#6-网关)
- [7. 配置中心](#7-配置中心)
- [8. 链路追踪](#8-链路追踪)
- [9. 踩坑与反模式](#9-踩坑与反模式)
- [10. 面试追问应对](#10-面试追问应对)

---

## 1. 整体架构概览

```
Spring Cloud 微服务典型架构：

  用户请求
    ↓
  网关层（Spring Cloud Gateway / Nginx）
    ↓ 路由 + 鉴权 + 限流
  服务调用（OpenFeign + LoadBalancer）
    ↓ 负载均衡
  服务注册中心（Nacos / Eureka）
    ↓ 服务发现
  业务服务（订单服务、库存服务、支付服务...）
    ↓ 熔断保护（Sentinel / Hystrix）
  基础设施（配置中心、链路追踪、监控告警）

版本演进：
  Spring Cloud Netflix（第一代）：Eureka + Ribbon + Hystrix + Feign + Zuul
  Spring Cloud Alibaba（第二代，国内主流）：Nacos + Sentinel + Seata + OpenFeign + Gateway
  Spring Cloud 2020+：Eureka 停更，LoadBalancer 替代 Ribbon，Gateway 替代 Zuul
```

---

## 2. 服务注册与发现

### 2.1 核心概念

```
为什么需要注册中心？
  单体架构：服务之间直接方法调用（IP:Port 写死在代码里）
  微服务架构：服务实例动态扩缩容，IP 会变，需要服务自动发现

注册中心的作用：
  1. 服务注册：服务启动时，把自己的 IP:Port 注册到注册中心
  2. 服务发现：消费者从注册中心获取服务实例列表
  3. 健康检查：定期检查服务是否存活，剔除死节点
```

### 2.2 选型对比

| 特性 | Eureka | Nacos | Consul | ZooKeeper |
|------|--------|-------|--------|-----------|
| 开发方 | Netflix | 阿里 | HashiCorp | Apache |
| 协议 | HTTP | HTTP + gRPC | HTTP + Raft | TCP + ZAB |
| 一致性 | AP（最终一致） | AP + CP 可选 | CP（强一致） | CP（强一致） |
| 健康检查 | 客户端心跳 | 客户端心跳 + 服务端探测 | 服务端探测 | 临时节点 + 会话 |
| 配置中心 | 不支持 | 内置支持 | 支持 | 不支持 |
| 负载均衡 | 需配合 Ribbon | 内置权重 | 需配合 | 需配合 |
| 多数据中心 | 支持（Region/AZ） | 支持 | 原生支持 | 需额外配置 |
| 社区活跃度 | 停更 | 活跃 | 活跃 | 活跃但偏底层 |
| 国内使用 | 存量项目 | 新项目主流 | 外企/云原生 | 底层协调 |

### 2.3 Eureka 详解

```
架构：
  Eureka Server：注册中心（可集群，Peer 复制）
  Eureka Client：服务提供者/消费者

注册流程：
  1. 服务启动 → 向 Eureka Server 注册（POST /apps/{appId}）
  2. 每 30s 发送心跳续约（PUT /apps/{appId}/{instanceId}）
  3. 每 30s 拉取服务列表（GET /apps）
  4. Eureka Server 90s 未收到心跳，剔除服务

自我保护机制：
  问题：网络分区时，大量服务心跳丢失，Eureka 误剔除健康服务
  解决：如果 15 分钟内心跳成功率 < 85%，进入自我保护模式
        → 不剔除任何服务（宁可保留死节点，也不误杀活节点）
  注意：生产环境建议开启，开发环境可关闭

缺点：
  1. 停更维护（Netflix 不再维护，Spring Cloud 2020+ 移除）
  2. 只支持 AP，无 CP 模式
  3. 无配置中心能力
```

### 2.4 Nacos 详解（国内主流）

```
架构：
  Nacos Server：注册中心 + 配置中心（可集群，支持 Derby/MySQL 持久化）
  Nacos Client：服务提供者/消费者

服务注册（临时实例 vs 永久实例）：
  临时实例（默认）：
    - 客户端心跳（5s 一次）
    - 服务端 15s 未收到心跳，剔除
    - 数据不持久化，Server 重启后需重新注册
  永久实例：
    - 服务端主动探测（HTTP/TCP/MySQL）
    - 适合非 JVM 服务（如 Nginx、MySQL）
    - 数据持久化到数据库

一致性模式：
  AP 模式（默认）：Distro 协议，最终一致，适合服务注册
  CP 模式：Raft 协议，强一致，适合配置中心

配置中心（Nacos Config）：
  - 支持动态配置刷新（@RefreshScope）
  - 支持多环境（Namespace：dev/test/prod）
  - 支持配置共享（shared-configs）
  - 支持配置监听（长轮询，1s 内感知变更）

面试要点：
  "Nacos 和 Eureka 的区别" → AP+CP 可选、内置配置中心、支持非临时实例、长轮询配置推送
```

---

## 3. 负载均衡

### 3.1 Ribbon（已停更）

```
Ribbon：客户端负载均衡器，集成在消费者端

工作方式：
  1. 从 Eureka 获取服务列表（如 order-service 有 3 个实例）
  2. 消费者本地维护服务列表
  3. 按策略选择一个实例调用

负载策略：
  RoundRobinRule：轮询（默认）
  RandomRule：随机
  WeightedResponseTimeRule：按响应时间加权，响应慢的权重低
  RetryRule：重试机制
  BestAvailableRule：选择并发请求最少的实例
  AvailabilityFilteringRule：过滤掉失败次数多的实例

缺点：
  - 与 Eureka 深度绑定
  - 停更维护
  - Spring Cloud 2020+ 被 LoadBalancer 替代
```

### 3.2 Spring Cloud LoadBalancer（推荐）

```
LoadBalancer：Spring Cloud 官方替代 Ribbon 的负载均衡器

特点：
  1. 与注册中心解耦（支持 Nacos、Eureka、Consul）
  2. 支持 Reactive 编程（WebFlux）
  3. 可自定义负载均衡策略

负载策略：
  RoundRobinLoadBalancer：轮询（默认）
  RandomLoadBalancer：随机
  WeightedLoadBalancer：权重（Nacos 支持）
 自定义：实现 ReactorServiceInstanceLoadBalancer

使用方式：
  @LoadBalancerClient("order-service")  // 声明式使用
  RestTemplate / WebClient 自动集成 LoadBalancer
```

---

## 4. 服务调用

### 4.1 OpenFeign

```
OpenFeign：声明式 HTTP 客户端，简化服务间调用

对比：
  RestTemplate：编程式，代码冗长
  WebClient：Reactive，但代码仍多
  OpenFeign：声明式，像调用本地方法一样调用远程服务

使用示例：
  @FeignClient(name = "order-service", fallback = OrderServiceFallback.class)
  public interface OrderServiceClient {
      @GetMapping("/orders/{orderId}")
      Order getOrder(@PathVariable("orderId") String orderId);
  }

  // 消费者直接注入使用
  @Autowired
  private OrderServiceClient orderServiceClient;

  Order order = orderServiceClient.getOrder("O123");

底层原理：
  1. @FeignClient 被解析后，生成动态代理（JDK Dynamic Proxy）
  2. 调用方法时，代理拦截，解析注解构建 HTTP 请求
  3. 通过 LoadBalancer 选择服务实例
  4. 发送 HTTP 请求，获取响应后反序列化

集成：
  - 集成 LoadBalancer：服务发现 + 负载均衡
  - 集成 Sentinel：熔断降级（@FeignClient fallback）
  - 集成 Hystrix：已废弃，改用 Sentinel
```

### 4.2 Feign 的性能优化

```
问题：OpenFeign 默认使用 HTTP 1.1 + 连接池，性能不如 RPC

优化方案：
  1. 连接池：
     Apache HttpClient / OKHttp 替换默认的 URLConnection
     feign.httpclient.enabled = true
     feign.okhttp.enabled = true

  2. 压缩：
     feign.compression.request.enabled = true
     feign.compression.response.enabled = true

  3. 超时配置：
     feign.client.config.default.connectTimeout = 2000
     feign.client.config.default.readTimeout = 5000

  4. 日志级别：
     NONE / BASIC / HEADERS / FULL
     生产环境用 BASIC，避免 FULL 打印 Body 影响性能

  5. 连接复用：
     HTTP Keep-Alive，减少 TCP 握手开销
```

---

## 5. 熔断与降级

### 5.1 Hystrix（已停更）

```
Hystrix：Netflix 开源的熔断降级库，已停更

核心概念：
  熔断（Circuit Breaker）：
    - 服务故障率超过阈值，自动打开熔断器
    - 后续请求直接失败，不再调用下游
    - 一段时间后进入半开状态，允许少量请求试探
    - 如果成功，关闭熔断器；如果失败，继续打开

  降级（Fallback）：
    - 熔断或异常时，返回默认值或缓存值

熔断状态机：
  CLOSED（关闭）→ 正常调用
    ↓ 失败率 > 阈值
  OPEN（打开）→ 直接失败，走 Fallback
    ↓ 经过休眠时间（如 5s）
  HALF_OPEN（半开）→ 允许少量请求试探
    ↓ 成功
  CLOSED（关闭）→ 恢复正常
    ↓ 失败
  OPEN（打开）→ 继续熔断

Hystrix 的局限：
  1. 基于线程池隔离，线程数有限，高并发下线程耗尽
  2. 停更维护，Spring Cloud 2020+ 移除
  3. 只支持同步调用，不支持 Reactive
```

### 5.2 Sentinel（阿里，国内主流）

```
Sentinel：阿里开源的流量控制、熔断降级、系统保护组件

对比 Hystrix：

  | 维度 | Hystrix | Sentinel |
  |------|---------|----------|
  | 隔离方式 | 线程池隔离 | 信号量隔离（更轻量） |
  | 熔断策略 | 基于失败率 | 基于失败率、慢调用比例、异常数 |
  | 限流 | 不支持 | 支持（滑动窗口、令牌桶、漏桶） |
  | 热点参数限流 | 不支持 | 支持 |
  | 系统保护 | 不支持 | 支持（CPU、Load、RT） |
  | 控制台 | 简单 Dashboard | 完善的可视化控制台 |
  | 维护状态 | 停更 | 活跃 |

Sentinel 的熔断策略：
  1. 慢调用比例：响应时间超过阈值的比例 > 设定值，触发熔断
  2. 异常比例：异常比例 > 设定值，触发熔断
  3. 异常数：异常数 > 设定值，触发熔断

Sentinel 的限流算法：
  1. 计数器（固定窗口）
  2. 滑动窗口（默认，更平滑）
  3. 令牌桶（允许突发）
  4. 漏桶（匀速输出）
  5. 热点参数限流（按参数限流，如按 user_id 限流）

使用方式：
  @SentinelResource(value = "getOrder", blockHandler = "getOrderBlockHandler")
  public Order getOrder(String orderId) {
      return orderService.getOrder(orderId);
  }

  public Order getOrderBlockHandler(String orderId, BlockException ex) {
      return new Order(); // 降级逻辑
  }
```

---

## 6. 网关

### 6.1 Spring Cloud Gateway（推荐）

```
Spring Cloud Gateway：Spring Cloud 官方网关，替代 Zuul

核心特性：
  1. 路由（Route）：根据路径、Header、Host 等匹配，转发到后端服务
  2. 断言（Predicate）：匹配条件（如 Path=/api/order/**）
  3. 过滤器（Filter）：请求/响应的加工处理

与 Zuul 的对比：
  | 维度 | Zuul 1.x | Zuul 2.x | Spring Cloud Gateway |
  |------|----------|----------|---------------------|
  | 底层 | Servlet 阻塞 | Netty 异步 | Netty 异步 |
  | 性能 | 一般 | 高 | 高 |
  | 编程模型 | 同步 | 异步 | Reactive |
  | 长连接支持 | 差 | 好 | 好 |
  | 维护状态 | 停更 | 未集成 Spring | 官方推荐 |

过滤器类型：
  GatewayFilter：路由级过滤器（如 StripPrefix、AddRequestHeader）
  GlobalFilter：全局过滤器（如鉴权、日志、限流）

常用过滤器：
  StripPrefix=1：去掉路径前缀（/api/order → /order）
  AddRequestHeader=X-Request-From, Gateway：添加请求头
  Retry：重试机制
  RequestRateLimiter：基于 Redis 的分布式限流
  CircuitBreaker：集成 Resilience4j 熔断

动态路由：
  从 Nacos/Consul 动态加载路由配置
  无需重启网关，路由变更实时生效
```

### 6.2 网关设计要点

```
网关的职责边界：
  应该做：
    1. 路由转发（路径映射到服务）
    2. 鉴权（JWT 校验、Token 刷新）
    3. 限流（API 级别、IP 级别）
    4. 日志/监控（请求日志、耗时统计）
    5. 协议转换（HTTP → gRPC / WebSocket）

  不应该做：
    1. 业务逻辑（如价格计算、库存判断）
    2. 复杂数据处理（大文件上传/下载）
    3. 长时间阻塞操作（如同步调用外部接口）

网关性能优化：
  1. 响应缓存：对不常变化的接口缓存响应
  2. 连接池：后端连接池复用，减少 TCP 握手
  3. 压缩：开启 gzip 压缩响应
  4. 异步化：Reactive 编程，非阻塞 I/O
  5. 集群化：多实例 + Nginx 负载均衡（LVS/SLB）
```

---

## 7. 配置中心

### 7.1 Spring Cloud Config（已淘汰）

```
Spring Cloud Config：Git 仓库作为配置存储

缺点：
  1. 无实时推送，需手动刷新或依赖 Webhook
  2. 无灰度发布能力
  3. 无权限管理
  4. 配置变更需改 Git，操作复杂

现状：已被 Nacos Config / Apollo 取代
```

### 7.2 Nacos Config（国内主流）

```
Nacos Config：配置管理 + 动态推送

核心特性：
  1. 动态配置刷新：
     @RefreshScope 注解的 Bean，配置变更后自动刷新
     无需重启服务

  2. 多环境隔离：
     Namespace：dev / test / prod
     Group：DEFAULT_GROUP / 自定义分组
     Data ID：{spring.application.name}-{profile}.properties

  3. 配置优先级：
     服务本地配置 < 共享配置（shared-configs）< 扩展配置（extension-configs）< 服务专属配置

  4. 配置监听：
     客户端长轮询（Long Polling）监听配置变更
     默认超时 30s，有变更立即返回

  5. 灰度发布：
     按 IP 灰度：先推送配置到部分实例
     按 Label 灰度：按标签分组推送

使用示例：
  bootstrap.yml:
    spring:
      cloud:
        nacos:
          config:
            server-addr: localhost:8848
            namespace: dev
            group: DEFAULT_GROUP
            file-extension: yaml
            shared-configs:
              - data-id: common.yaml
                group: DEFAULT_GROUP
                refresh: true
```

---

## 8. 链路追踪

```
Spring Cloud Sleuth + Zipkin：分布式链路追踪

核心概念：
  Trace：一次完整的请求链路（如用户下单）
  Span：链路中的每个节点（如网关调用 → 订单服务 → 库存服务）
  Trace ID：全局唯一，贯穿整个链路
  Span ID：每个 Span 的唯一标识
  Parent Span ID：标识父子关系

传播方式：
  HTTP 请求头中携带 Trace 信息：
    X-B3-TraceId：1234567890abcdef
    X-B3-SpanId：abcdef1234567890
    X-B3-ParentSpanId：0000000000000000
    X-B3-Sampled：1（是否采样）

集成：
  Sleuth：自动埋点，生成 Trace/Span
  Zipkin：可视化展示链路（调用关系、耗时、错误）
  或：SkyWalking、Jaeger（OpenTelemetry 标准）

采样策略：
  ProbabilityBasedSampler：按概率采样（如 10%）
  RateLimitingSampler：限制每秒采样数
  生产环境建议 1%~10% 采样，避免性能影响

注意：
  Sleuth 在 Spring Cloud 2022+ 中已被 Micrometer Tracing 替代
  新趋势：OpenTelemetry + Jaeger，统一标准
```

---

## 9. 踩坑与反模式

### 坑 1：用 Eureka 做新项目的注册中心
```
问题：Eureka 已停更，社区不再维护，Bug 无人修复
解决：新项目用 Nacos 或 Consul
```

### 坑 2：Feign 调用不用 Fallback
```
问题：下游服务挂了，Feign 调用直接抛异常，级联失败
解决：@FeignClient 必须配置 fallback，降级返回默认值或缓存
```

### 坑 3：Hystrix 线程池配置不合理
```
问题：Hystrix 线程池默认 10 个线程，高并发下线程耗尽
解决：改用 Sentinel（信号量隔离，更轻量），或调整 Hystrix 线程池大小
```

### 坑 4：Gateway 做太多业务逻辑
```
问题：网关中写了大量业务代码（如价格计算、优惠判断）
  → 网关变重，发布频率高，影响所有服务
解决：网关只做通用能力（路由、鉴权、限流），业务逻辑下沉到服务
```

### 坑 5：Nacos 配置变更没刷新
```
问题：修改了 Nacos 配置，但应用没生效
  → 忘记加 @RefreshScope
  → 或 Bean 不是 Spring 管理的
解决：需要动态刷新的 Bean 加 @RefreshScope，配置类加 @Configuration
```

### 坑 6：Feign 默认超时太长
```
问题：Feign 默认连接超时 10s，读取超时 60s
  → 下游服务卡死，消费者线程阻塞
解决：设置合理超时（连接 2s，读取 5s），配合熔断快速失败
```

### 坑 7：链路追踪没有采样，全量采集性能崩
```
问题：生产环境 100% 采样，Trace 数据量巨大，存储和查询慢
解决：生产环境按 1%~10% 采样，问题排查时临时提高采样率
```

---

## 10. 面试追问应对

**Q1：Spring Cloud 和 Spring Cloud Alibaba 有什么区别？**
> Spring Cloud Netflix 是一代微服务组件（Eureka + Ribbon + Hystrix + Zuul），但 Netflix 已停更维护。Spring Cloud Alibaba 是阿里开源的替代方案，用 Nacos 替代 Eureka 和 Config，用 Sentinel 替代 Hystrix，用 Seata 处理分布式事务，用 Gateway 替代 Zuul。目前国内新项目基本都是 Spring Cloud Alibaba 技术栈。

**Q2：Nacos 和 Eureka 的区别？**
> 1. Nacos 支持 AP 和 CP 两种模式（Eureka 只支持 AP）；2. Nacos 内置配置中心（Eureka 没有）；3. Nacos 支持非临时实例（服务端主动探测，适合非 JVM 服务）；4. Nacos 配置推送用长轮询，实时性更好；5. Nacos 社区活跃，Eureka 已停更。如果面试问到，优先选 Nacos。

**Q3：Feign 的底层原理是什么？**
> Feign 是声明式 HTTP 客户端。运行时，@FeignClient 注解的接口会被解析，生成 JDK 动态代理。调用方法时，代理拦截，解析方法上的注解（如 @GetMapping），构建 HTTP 请求。然后通过 LoadBalancer 从注册中心获取服务实例列表，选择一个实例发送请求。收到响应后，反序列化为返回值类型。Feign 底层集成了 Ribbon/LoadBalancer 做负载均衡，集成 Sentinel 做熔断。

**Q4：Sentinel 和 Hystrix 的区别？**
> 1. 隔离方式：Hystrix 用线程池隔离，Sentinel 用信号量隔离（更轻量）；2. 熔断策略：Hystrix 只支持失败率，Sentinel 支持失败率、慢调用比例、异常数；3. 限流：Hystrix 不支持限流，Sentinel 支持多种限流算法（滑动窗口、令牌桶、热点参数限流）；4. 系统保护：Sentinel 支持系统级保护（CPU、Load）；5. 控制台：Sentinel 有完善的可视化控制台；6. 维护状态：Hystrix 停更，Sentinel 活跃。

**Q5：Gateway 和 Nginx 的区别？**
> Gateway 是应用层网关，集成在 Spring Cloud 生态中，擅长服务路由、鉴权、限流、熔断，与注册中心、配置中心天然集成。Nginx 是网络层网关（反向代理），性能更高，擅长静态资源、SSL 终止、负载均衡。实际架构中通常两者共存：Nginx 在最外层（L7 负载均衡 + 静态资源），Gateway 在内部（微服务路由 + 业务鉴权）。

**Q6：Spring Cloud Gateway 的路由怎么配置？**
> 两种方式：1. YAML 静态配置（routes 数组，每个路由包含 id、uri、predicates、filters）；2. 动态路由（从 Nacos/Consul 加载，变更实时生效）。路由匹配时按顺序匹配，第一个匹配的路由生效。常用 Predicate 有 Path、Header、Query、Cookie 等，常用 Filter 有 StripPrefix（去前缀）、AddRequestHeader（加请求头）、Retry（重试）、RequestRateLimiter（限流）、CircuitBreaker（熔断）。

---

## 关联阅读

- [00-分布式系统核心设计模式](./00-分布式系统核心设计模式.md) — 分布式 ID、CAP、限流、降级等通用模式
- [02-数据一致性/04-分布式事务与Seata](../02-数据一致性/04-分布式事务与Seata.md) — Seata 与 Spring Cloud 集成
- [05-稳定性与可观测/01-链路追踪深度实战](../05-稳定性与可观测/01-链路追踪深度实战.md) — 链路追踪深度展开

---

> **一句话总结**：Spring Cloud 生态 = Nacos（注册+配置）+ OpenFeign（调用）+ LoadBalancer（负载均衡）+ Sentinel（熔断限流）+ Gateway（网关）+ Sleuth/Zipkin（链路追踪）。国内面试必须掌握这套技术栈，且要知道 Netflix 组件已停更、Nacos 和 Sentinel 是主流。
