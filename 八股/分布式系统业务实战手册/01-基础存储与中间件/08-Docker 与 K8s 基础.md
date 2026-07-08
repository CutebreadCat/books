# Docker 与 K8s 基础

> Docker 和 Kubernetes 是云原生时代的标配。面试中常问容器原理、Dockerfile 优化、K8s 核心对象和调度机制。本篇覆盖从容器隔离原理到 K8s 生产运维的完整知识链。

---

## 目录

- [1. 容器核心原理](#1-容器核心原理)
- [2. Docker 实践](#2-docker-实践)
- [3. Kubernetes 核心对象](#3-kubernetes-核心对象)
- [4. K8s 调度与调度器](#4-k8s-调度与调度器)
- [5. K8s 运维机制](#5-k8s-运维机制)
- [6. 生产踩坑](#6-生产踩坑)
- [7. 面试追问应对](#7-面试追问应对)

---

## 1. 容器核心原理

### 1.1 容器 vs 虚拟机

```
虚拟机（VM）：
  物理机 → Hypervisor → 完整 Guest OS → 应用 + 依赖
  资源占用：每个 VM 需要完整 OS，内存占用 GB 级
  启动速度：分钟级
  隔离级别：强隔离（硬件级别）

容器（Container）：
  物理机 → Host OS → Docker Engine → 容器（共享内核）→ 应用 + 依赖
  资源占用：共享 Host OS 内核，内存占用 MB 级
  启动速度：秒级
  隔离级别：进程级隔离（Namespace + Cgroups）

本质区别：
  VM 虚拟化硬件，容器虚拟化操作系统
  容器不是"轻量 VM"，而是"受限制的进程"
```

### 1.2 Namespace（命名空间）

```
Namespace：实现资源隔离，让容器中的进程"看不到"其他进程

六种 Namespace：
  PID Namespace：进程 ID 隔离（容器内 PID 1 是应用进程，宿主 PID 1 是 systemd）
  NET Namespace：网络隔离（容器有独立网卡、IP、路由表）
  MNT Namespace：挂载隔离（容器有独立的文件系统视图）
  UTS Namespace：主机名隔离（容器可设独立 hostname）
  IPC Namespace：进程间通信隔离（隔离共享内存、信号量）
  USER Namespace：用户隔离（容器 root 映射到宿主普通用户，安全隔离）

查看容器 Namespace：
  ls -l /proc/<pid>/ns/
  每个容器进程有独立的 ns 链接

面试要点：
  "容器和虚拟机的区别" → 容器共享内核，Namespace 做进程隔离，Cgroups 做资源限制
```

### 1.3 Cgroups（控制组）

```
Cgroups：限制容器能使用的资源（CPU、内存、磁盘 I/O、网络）

子系统（Subsystem）：
  cpu：限制 CPU 使用率（cfs_quota_us / cfs_period_us）
  cpuacct：统计 CPU 使用
  memory：限制内存（limit_in_bytes、swappiness）
  blkio：限制块设备 I/O
  devices：控制设备访问权限
  net_cls：标记网络包，用于 QoS

查看 Cgroups：
  cat /sys/fs/cgroup/memory/docker/<container_id>/memory.limit_in_bytes
  cat /sys/fs/cgroup/cpu/docker/<container_id>/cpu.cfs_quota_us

面试要点：
  "Docker 如何限制容器资源" → Cgroups，限制 CPU（quota/period）、内存（limit）、I/O（blkio）
```

### 1.4 UnionFS（联合文件系统）

```
UnionFS：实现镜像分层，多个只读层 + 一个可写层叠加

Docker 镜像分层：
  Base Image（基础镜像层）：如 ubuntu:20.04
  → 只读，可被多个镜像共享
  Add Dependency Layer（依赖层）：如 apt-get install xxx
  → 只读
  Application Layer（应用层）：如 COPY app.jar
  → 只读
  Container Layer（容器层）：运行时写入
  → 可写

写时复制（Copy-on-Write）：
  修改文件时，从只读层复制到可写层，只读层不变
  优点：镜像共享、节省空间、快速启动

面试要点：
  "Docker 镜像为什么分层" → 复用基础层、减少存储、加速构建和传输
  "容器写文件会不会影响镜像" → 不会，写时复制到容器层，镜像层只读
```

---

## 2. Docker 实践

### 2.1 Dockerfile 优化

```
优化原则：减少层数、减小体积、加速构建

1. 使用多阶段构建（Multi-stage）：
   构建阶段（含编译工具、源码）→ 运行阶段（仅运行时 + 产物）
   例：
     FROM maven:3.8-openjdk-17 AS build
     COPY src /app/src
     RUN mvn -f /app/pom.xml clean package

     FROM eclipse-temurin:17-jre-alpine
     COPY --from=build /app/target/app.jar /app.jar
     ENTRYPOINT ["java","-jar","/app.jar"]
   效果：镜像从 700MB → 80MB

2. 选择轻量基础镜像：
   alpine 比 ubuntu 小 90%
   但注意：alpine 用 musl libc，某些 Java 应用可能有兼容问题
   推荐：eclipse-temurin:17-jre-alpine 或 distroless

3. 合并 RUN 命令：
   每个 RUN 创建一层，合并减少层数
   RUN apt-get update && apt-get install -y xxx && rm -rf /var/lib/apt/lists/*

4. 利用缓存：
   不常变的指令放前面（如依赖安装）
   常变的指令放后面（如 COPY 源码）
   这样只有最后一层重新构建

5. .dockerignore：
   排除 .git、target/、*.md 等不需要的文件
```

### 2.2 容器运行时与镜像安全

```
容器运行时演进：
  runc：Docker 底层标准运行时（OCI 标准）
  containerd：Docker 抽出的容器管理引擎（镜像、容器生命周期）
  CRI-O：K8s 的轻量运行时，直接用 runc

镜像安全：
  1. 使用非 root 用户运行：
     RUN addgroup -S appgroup && adduser -S appuser -G appgroup
     USER appuser
  2. 扫描镜像漏洞：Trivy、Clair、Snyk
  3. 最小权限原则：只暴露必要端口，只挂载必要目录
  4. 镜像签名：Docker Content Trust（DCT）/ Notary

面试要点：
  "Docker 和 containerd 的关系" → containerd 是 Docker 的容器管理引擎，
  Docker 是一个完整平台，containerd 是其中的运行时管理组件，K8s 1.24+ 直接对接 containerd
```

---

## 3. Kubernetes 核心对象

### 3.1 K8s 架构

```
Master 节点（控制平面）：
  API Server：所有操作的入口，REST API，认证鉴权
  etcd：分布式键值存储，存储所有集群状态
  Scheduler：调度器，决定 Pod 调度到哪个 Node
  Controller Manager：运行各种控制器（Deployment、ReplicaSet、Node）

Worker 节点：
  kubelet：管理节点上的 Pod 生命周期
  kube-proxy：实现 Service 的网络代理和负载均衡
  Container Runtime：containerd / CRI-O / Docker

etcd 的作用：
  存储所有 K8s 对象（Pod、Service、ConfigMap 等）
  存储集群状态（节点信息、资源分配）
  Raft 协议保证一致性（CP 系统）
  所有组件通过 API Server 读写 etcd
```

### 3.2 Pod（最小调度单元）

```
Pod 的本质：
  不是容器，是容器的"组"（一个 Pod 可包含多个容器）
  一个 Pod 内的容器共享 Network Namespace（IP、端口空间）
  共享存储卷（Volume）
  通常 1 Pod = 1 主容器 + 0~N 个 Sidecar（如日志采集、监控）

Pause 容器（Infra 容器）：
  每个 Pod 第一个创建的容器，持有 Pod 的 Network Namespace
  其他容器加入这个 Namespace，共享 IP 和端口空间
  如果 Pause 容器挂了，整个 Pod 重启

Pod 生命周期：
  Pending → Running → Succeeded / Failed
  当 Pod 被调度但容器还没启动 → Pending
  当至少一个容器运行 → Running
  所有容器正常退出 → Succeeded
  至少一个容器异常退出 → Failed

面试要点：
  "为什么 Pod 是最小调度单元，而不是容器" →
  1. 需要协同调度（主容器 + Sidecar 必须同节点）
  2. 需要共享资源（网络、存储）
  3. 需要统一生命周期管理
```

### 3.3 Deployment 与 ReplicaSet

```
ReplicaSet（RS）：
  确保指定数量的 Pod 副本始终运行
  通过 label selector 管理 Pod
  如果 Pod 挂了，自动创建新 Pod
  如果 Pod 多了，自动删除多余 Pod

Deployment：
  在 ReplicaSet 之上，提供声明式更新和回滚
  核心功能：滚动更新（Rolling Update）

滚动更新机制：
  1. 创建新的 ReplicaSet（新版本的 Pod）
  2. 逐步增加新 ReplicaSet 副本数，减少旧 ReplicaSet 副本数
  3. 通过 maxSurge（最大超出副本数）和 maxUnavailable（最大不可用副本数）控制节奏
  4. 旧 ReplicaSet 保留，支持回滚

回滚：
  kubectl rollout undo deployment/my-app
  回滚到上一个版本（Deployment 会保留历史 ReplicaSet）
```

### 3.4 Service 与网络

```
Service：为一组 Pod 提供稳定的网络入口（IP + 端口 + 负载均衡）

Service 类型：
  ClusterIP（默认）：集群内部访问，分配集群内 IP
  NodePort：暴露节点端口（30000-32767），外部可访问
  LoadBalancer：云厂商负载均衡（如 AWS ELB、阿里云 SLB）
  ExternalName：DNS 别名，指向外部服务

Service 网络原理：
  kube-proxy 在节点上维护 iptables / IPVS 规则
  访问 ClusterIP 时，iptables 将流量 DNAT 到后端 Pod IP
  IPVS 模式比 iptables 性能更好（大规模集群）

Headless Service：
  clusterIP: None，不分配虚拟 IP
  DNS 直接返回后端 Pod IP（用于有状态服务，如 StatefulSet）

Ingress：
  7 层（HTTP）路由，基于域名和路径转发
  需要 Ingress Controller（如 Nginx Ingress、Traefik）
  对比 Service：Service 是 4 层负载均衡，Ingress 是 7 层

面试要点：
  "Service 的几种类型和适用场景" →
  ClusterIP：内部服务调用；NodePort：开发测试；LoadBalancer：生产暴露；ExternalName：外部服务代理
  "kube-proxy 的三种模式" → userspace（已淘汰）、iptables（默认）、IPVS（高性能，推荐）
```

### 3.5 ConfigMap 与 Secret

```
ConfigMap：存储非敏感配置（如配置文件、命令行参数）
  使用方式：环境变量注入、Volume 挂载为文件、命令行参数

Secret：存储敏感数据（如密码、Token、证书）
  默认只 base64 编码，不加密！（需配合 KMS 或 etcd 加密）
  使用方式：挂载为文件、环境变量注入（不推荐，容易泄露）
  最佳实践：用 Vault / Sealed Secrets 管理密钥

注意：
  ConfigMap/Secret 变更后，已挂载的 Pod 不会自动更新
  需要重启 Pod 或手动触发更新（除非用 Reloader 等工具）
```

### 3.6 StatefulSet（有状态服务）

```
StatefulSet：管理有状态应用（如 MySQL、Redis、Kafka）

与 Deployment 的区别：
  | 特性 | Deployment | StatefulSet |
  |------|-----------|-------------|
  | Pod 命名 | 随机（web-5d4f8b7c9-x2z1a） | 有序（web-0, web-1, web-2） |
  | 创建/删除 | 并行 | 按顺序 |
  | 存储 | 共享存储（所有 Pod 共享 PVC） | 独立存储（每个 Pod 有自己的 PVC） |
  | 网络 | 通过 Service 访问 | 固定 DNS（web-0.mysql） |
  | 使用场景 | 无状态（Web、API） | 有状态（数据库、消息队列） |

存储：
  volumeClaimTemplates：每个 Pod 自动创建独立 PVC
  Pod 重建后，重新绑定原来的 PVC，数据不丢失

面试要点：
  "为什么有状态服务用 StatefulSet 不用 Deployment" →
  1. 需要稳定网络标识（固定 DNS）
  2. 需要独立存储（每个实例有自己的数据）
  3. 需要有序部署和扩容（防止脑裂、数据不一致）
```

---

## 4. K8s 调度与调度器

### 4.1 调度流程

```
Pod 调度过程：
  1. 用户创建 Pod → 写入 etcd（Pending 状态）
  2. Scheduler 监听 Pod 创建事件
  3. 调度阶段：
     a. 过滤（Predicates）：排除不满足条件的节点
        - 资源是否充足（CPU、内存、GPU）
        - 节点是否有污点（Taint），Pod 是否容忍（Toleration）
        - 节点亲和性/反亲和性（Node Affinity / Anti-affinity）
        - Pod 亲和性/反亲和性
     b. 打分（Priorities）：对剩余节点打分，选最高分
        - 资源均衡（LeastRequestedPriority：优先选资源多的节点）
        - 节点亲和性权重
        - Pod 反亲和性（尽量分散）
  4. 绑定（Bind）：将 Pod 绑定到选定节点，写入 etcd
  5. kubelet 监听 Pod 绑定 → 在节点上创建容器

调度器扩展：
  Scheduler Framework：支持自定义插件（过滤插件、打分插件、绑定插件）
  调度器可配置多个（不同业务用不同调度器）
```

### 4.2 亲和性与反亲和性

```
节点亲和性（Node Affinity）：
  Pod 倾向于调度到满足特定条件的节点
  如： Pod 需要 SSD 磁盘 → 调度到带 disk=ssd 标签的节点
  requiredDuringSchedulingIgnoredDuringExecution：硬约束（必须满足）
  preferredDuringSchedulingIgnoredDuringExecution：软约束（尽量满足）

Pod 反亲和性（Pod Anti-Affinity）：
  Pod 尽量不和其他 Pod 调度到同一节点
  如：同一服务的 Pod 分散到不同节点，避免单节点故障导致全挂

污点与容忍（Taint & Toleration）：
  Taint：节点属性，排斥 Pod（如 dedicated=gpu:NoSchedule）
  Toleration：Pod 属性，容忍特定 Taint（如允许调度到 GPU 节点）
  用途：专用节点（GPU、SSD）、节点维护（NoSchedule、NoExecute）

面试要点：
  "如何让 Pod 尽量分散到不同节点" → Pod Anti-Affinity + topologyKey: kubernetes.io/hostname
  "如何让 Pod 只调度到 SSD 节点" → Node Affinity 或 Taint + Toleration
```

---

## 5. K8s 运维机制

### 5.1 健康检查（Probe）

```
三种探针：
  Liveness Probe（存活探针）：
    - 检测容器是否还活着
    - 失败 → kubelet 重启容器
    - 适用：容器死锁、死循环等应用级故障

  Readiness Probe（就绪探针）：
    - 检测容器是否准备好接收流量
    - 失败 → Pod 从 Service Endpoints 中移除（不再接收流量）
    - 适用：应用启动慢，启动完成前不接收流量；应用临时不可用（如数据库连接断开）

  Startup Probe（启动探针）：
    - 检测容器是否启动完成
    - 失败 → kubelet 重启容器
    - 适用：启动时间长的应用（如 JVM 应用），避免 Liveness 误判

探针方式：
  HTTPGet：HTTP 请求，状态码 2xx/3xx 为成功
  TCPSocket：TCP 端口是否可连接
  Exec：执行命令，退出码 0 为成功

面试要点：
  "Liveness 和 Readiness 的区别" →
  Liveness 失败 → 重启容器（解决自愈）；Readiness 失败 → 从 Service 摘除（解决流量问题）
  不要同时设置 Liveness 和 Readiness 为相同配置，否则启动慢的应用会被反复重启
```

### 5.2 资源管理

```
资源请求与限制：
  requests：Pod 申请的资源（调度时保证节点有 >= requests 的资源）
  limits：Pod 允许使用的最大资源（运行时限制，Cgroups 实现）

QoS 等级（由 requests 和 limits 决定）：
  Guaranteed：requests == limits，且只设置 limits（最高优先级，OOM 时最后被杀）
  Burstable：requests < limits（大多数应用，中等优先级）
  BestEffort：不设置 requests 和 limits（最低优先级，OOM 时最先被杀）

LimitRange：命名空间级别，限制 Pod 的 requests/limits 默认值和范围
ResourceQuota：命名空间级别，限制资源总使用量（如最多 100 CPU，200GB 内存）

面试要点：
  "QoS 三种级别和 OOM 优先级" → Guaranteed > Burstable > BestEffort，OOM 时 BestEffort 先被杀
  "为什么不设置 limits" → 不设置 limits 可能耗尽节点资源，影响其他 Pod；不设置 requests 可能调度到资源不足节点
```

### 5.3 HPA 与 VPA

```
HPA（Horizontal Pod Autoscaler）：水平自动扩缩容
  根据指标（CPU 利用率、内存、自定义指标）自动调整 Pod 副本数
  公式：期望副本数 = ceil(当前指标 / 目标指标 * 当前副本数)
  注意：需要配置 requests 才能计算 CPU 利用率

VPA（Vertical Pod Autoscaler）：垂直自动扩缩容
  自动调整 Pod 的 requests/limits（CPU、内存）
  有"Initial"（仅初始化时）和"Auto"（重建 Pod）两种模式
  生产环境 VPA 和 HPA 不建议同时用（可能冲突）

CronHPA：定时扩缩容（如每天 9 点扩容，18 点缩容）

Cluster Autoscaler：集群级别，根据 Pod 无法调度（Pending）自动扩容节点
```

### 5.4 网络模型

```
K8s 网络要求：
  1. 所有 Pod 可直接通信（不需要 NAT）
  2. 所有节点可直接通信（不需要 NAT）
  3. Pod 看到的 IP 和其他 Pod 看到的一样（IP 直通）

CNI 插件：
  Flannel：简单 overlay 网络，UDP/VXLAN 封装，适合中小集群
  Calico：支持 BGP 路由（underlay 和 overlay），支持网络策略（NetworkPolicy），适合大规模生产
  Cilium：基于 eBPF，高性能，支持 L3/L4/L7 网络策略，适合云原生安全
  Weave：自动发现，自动加密

NetworkPolicy：
  定义 Pod 级别的网络访问规则（入站/出站）
  默认所有 Pod 之间互通，NetworkPolicy 可以限制
  需要 CNI 插件支持（Calico、Cilium 支持，Flannel 不支持）

面试要点：
  "Flannel 和 Calico 的区别" →
  Flannel 简单 overlay（VXLAN），适合中小集群；Calico 支持 BGP underlay（性能更好），支持 NetworkPolicy，适合大规模生产
  "K8s 网络为什么要求 Pod 直接互通" → 微服务架构需要任意服务间直接调用，不需要 NAT 转换
```

---

## 6. 生产踩坑

### 坑 1：镜像拉取策略不当导致启动慢
```
问题：imagePullPolicy: Always，每次启动都拉取镜像，内网慢或外部镜像仓库宕机导致 Pod 起不来
解决：生产环境用 IfNotPresent，镜像 tag 用固定版本（不用 latest）
```

### 坑 2：不设置 resources 导致节点资源耗尽
```
问题：Pod 不设置 requests/limits，某个 Pod 内存泄漏耗尽节点，导致所有 Pod 被 OOM Kill
解决：所有 Pod 必须设置 requests 和 limits，命名空间配 ResourceQuota
```

### 坑 3：Liveness 配置不当导致反复重启
```
问题：Liveness 探针间隔太短，应用启动慢但探针已认为失败，反复重启
解决：启动慢的应用用 Startup Probe，Liveness 设置合理间隔和失败阈值
```

### 坑 4：StatefulSet 误用 Deployment 的 RollingUpdate
```
问题：数据库用 Deployment，滚动更新时旧 Pod 还没停，新 Pod 已启动，导致脑裂
解决：有状态服务用 StatefulSet，有序部署，配合 PodDisruptionBudget 控制
```

### 坑 5：Secret 只 base64 编码，不安全
```
问题：Secret 的值只是 base64，etcd 中明文存储，有权限的人可以直接查看
解决：启用 etcd 加密（--encryption-provider-config），或用 Vault 管理密钥
```

### 坑 6：ConfigMap 更新后 Pod 不刷新
```
问题：修改 ConfigMap 后，已挂载的 Pod 配置文件没更新
解决：使用 Reloader（监听 ConfigMap/Secret 变更，自动滚动更新 Pod）
```

### 坑 7：HPA 不生效，因为没有配 requests
```
问题：配置了 HPA 但 Pod 不扩缩容
  → 因为 Pod 没有设置 CPU requests，HPA 无法计算利用率
解决：所有 Pod 必须设置 resources.requests.cpu
```

---

## 7. 面试追问应对

**Q1：Docker 和容器的关系是什么？**
> Docker 是一个容器平台（包含客户端、守护进程、镜像管理、容器运行时）。容器是一种轻量级的虚拟化技术，通过 Linux Namespace 做隔离、Cgroups 做资源限制。Docker 让容器的创建、分发、运行变得简单。现在容器运行时标准由 OCI（Open Container Initiative）定义，containerd、CRI-O 都是符合 OCI 的运行时，Docker 也基于 containerd。

**Q2：Namespace 和 Cgroups 的区别？**
> Namespace 负责"隔离"（进程看不到外面的资源），Cgroups 负责"限制"（进程能使用多少资源）。比如一个容器：用 PID Namespace 隔离进程（容器内 PID 1 是应用），用 Cgroups 限制只能使用 2 核 CPU 和 4GB 内存。两者配合实现了容器的隔离和资源控制。

**Q3：K8s 的 Pod 为什么要设计成这样？一个 Pod 里放多个容器是什么场景？**
> Pod 是 K8s 的最小调度单元，因为某些应用需要多个容器协同工作（如主应用 + 日志采集 Sidecar）。Pod 内的容器共享网络（同一 IP）、共享存储（Volume），可以紧密协作。常见场景：1）主容器 + Sidecar（如日志采集、监控代理）；2）主容器 + 初始化容器（Init Container，按顺序执行初始化任务）；3）主容器 + 代理容器（如 Envoy Sidecar，服务网格）。

**Q4：Deployment 的滚动更新是怎么实现的？**
> Deployment 通过 ReplicaSet 管理 Pod。滚动更新时，Deployment 创建一个新的 ReplicaSet（新版本），然后逐步增加新 ReplicaSet 的副本数，减少旧 ReplicaSet 的副本数，通过 maxSurge（最大超出的副本数）和 maxUnavailable（最大不可用的副本数）控制节奏。旧的 ReplicaSet 保留，如果新版本有问题，可以通过 kubectl rollout undo 回滚。

**Q5：Service 的 ClusterIP 是怎么实现的？**
> Service 的 ClusterIP 是一个虚拟 IP，不对应任何实际网卡。kube-proxy 在每个节点上维护 iptables 或 IPVS 规则，访问 ClusterIP 时，iptables 将流量 DNAT 到后端 Pod 的真实 IP。如果 Service 有多个后端 Pod，iptables 会按概率或轮询分发。IPVS 模式比 iptables 性能更好，适合大规模集群。

**Q6：HPA 的原理是什么？为什么不直接用 VPA？**
> HPA 通过 Metrics Server 获取 Pod 的 CPU/内存利用率，根据公式"期望副本数 = ceil(当前指标 / 目标指标 * 当前副本数)"自动调整副本数。HPA 是水平扩容（增加 Pod 数量），VPA 是垂直扩容（增加单 Pod 资源）。HPA 更适合无状态服务，因为增加副本数可以快速响应流量变化；VPA 需要重建 Pod，不适合频繁调整。两者不建议同时用，因为可能冲突（HPA 扩容时 VPA 缩容资源）。

**Q7：K8s 的 etcd 如果挂了会怎样？怎么保证高可用？**
> etcd 是 K8s 的数据存储层，所有对象（Pod、Service、Deployment）和状态都存 etcd。如果 etcd 挂了，API Server 无法读写数据，整个集群无法创建、更新、删除资源（但已运行的 Pod 不受影响）。etcd 通过 Raft 协议实现高可用，推荐部署 3 或 5 节点 etcd 集群。生产环境 etcd 磁盘必须用 SSD，网络延迟要低，否则影响 K8s 整体性能。

---

## 关联阅读

- [03-分布式架构与设计/03-Spring Cloud 生态](../03-分布式架构与设计/03-Spring%20Cloud%20生态.md) — 微服务架构与 K8s 容器化的关系
- [03-分布式架构与设计/00-分布式系统核心设计模式](../03-分布式架构与设计/00-分布式系统核心设计模式.md) — 分布式 ID、CAP、限流等通用模式
- [05-稳定性与可观测/01-链路追踪深度实战](../05-稳定性与可观测/01-链路追踪深度实战.md) — 容器环境的链路追踪

---

> **一句话总结**：Docker = Namespace（隔离）+ Cgroups（限制）+ UnionFS（分层）。K8s = Pod（调度单元）+ Deployment（无状态管理）+ StatefulSet（有状态管理）+ Service（网络入口）+ HPA（自动扩缩）。面试重点：容器原理、Pod 设计、Deployment 滚动更新、Service 网络实现、调度机制。
