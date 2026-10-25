# Datadog Agent 和 OTel Collector DaemonSet，究竟有多像？

> 2026-10-25 · Kubernetes 部署架构调研  
> 阅读目标：理解两者在 Kubernetes 中如何部署、Trace 怎样流转，以及 Spanloom 是否需要复制这一层。

## 先说结论

**可以把 Datadog Agent 理解成“Datadog 体系里的节点级 Collector”。** 在 Kubernetes 中，两者都能以 **DaemonSet** 运行：每个 Node 部署一份，接收这个节点上应用产生的数据，再发送到后端。

但这个类比只适用于**部署位置和基本职责**。Datadog Trace Agent 除了接收和转发 Span，还会执行 Datadog 特有的数据处理、采样和统计聚合。OpenTelemetry Collector 是可配置的数据处理框架，实际具备什么功能，由组件和 Pipeline 决定。

> **部署上很像；内部实现和数据语义不同。**

## 1. 从一条请求的 Trace 看两种部署

假设 Kubernetes 集群里运行着几个 Java 服务，各自在自己的 Pod 中。Java Agent 负责在应用运行时生成 Span，部署在节点上的采集进程则接收并转发这些数据。

### Datadog 的典型链路

~~~mermaid
flowchart LR
    subgraph N["Kubernetes Node"]
        J["业务 Pod<br/>dd-java-agent"] -->|"Trace API"| A["Datadog Agent<br/>DaemonSet Pod"]
    end
    A -->|"HTTPS"| D["Datadog Intake"]
    D --> B["APM 后端"]
~~~

在 Datadog 方案中，请求经过 **dd-java-agent** 自动埋点后，由 Tracer 将 Trace 发送给节点上的 Datadog Agent。Agent 内负责 APM 的 **Trace Agent** 接收数据，处理后发送到 Datadog Intake。

这里容易混淆的是两个“Agent”：**dd-java-agent 在 JVM 内生成 Span；Datadog Agent 是 JVM 外的独立进程**，通常通过 Kubernetes DaemonSet 部署。Trace Agent 是后者负责 APM Trace 的子系统。

Datadog 官方 [Helm Chart README](https://github.com/DataDog/helm-charts/blob/main/charts/datadog/README.md) 明确说明，默认在集群节点上通过 DaemonSet 部署 Agent。

### OTel Collector 的对应链路

~~~mermaid
flowchart LR
    subgraph N["Kubernetes Node"]
        J["业务 Pod<br/>OTel Java Agent"] -->|"OTLP"| A["OTel Collector<br/>DaemonSet Pod"]
    end
    A --> E["OTLP 后端 / Kafka / Gateway"]
~~~

OTel Java Agent 同样在 JVM 中生成 Span，通过 OTLP 发送到节点上的 Collector。Collector 再按照配置执行接收、处理和导出。

这就是两者架构相似的地方：**Node 上可以运行多个业务 Pod，但通常只部署一份 DaemonSet 采集进程**。业务 Pod 不需要各自启动一个 Collector。

区别是，OTel 可以完全不部署节点级 Collector，让应用直接发送到 **Gateway** 或支持的后端。**DaemonSet 是一种部署选择，不是 OTLP 的强制环节。**

参考：[OTel Collector Agent 模式](https://opentelemetry.io/docs/collector/deploy/agent/) · [Gateway 模式](https://opentelemetry.io/docs/collector/deploy/gateway/)。

### 一张表看对应关系

| 问题 | Datadog | OpenTelemetry |
|---|---|---|
| JVM 里谁生成 Span？ | dd-java-agent / Tracer | OTel Java Agent / SDK |
| Node 上部署什么？ | Datadog Agent | Collector Agent |
| Kubernetes 常用的部署方式？ | DaemonSet（官方 Helm 默认） | DaemonSet（Agent 模式） |
| 应用如何发送数据？ | Datadog Trace API 等 | 通常使用 OTLP |
| 采集后发往哪里？ | Datadog Intake | 按 Exporter 配置：OTLP、Kafka 等 |
| 必须部署 Node 采集层吗？ | Datadog 典型 APM 架构使用，也有特定直连方式 | **不必须**，可以用 Gateway-only |

**一个实际部署细节**：DaemonSet Pod 和业务 Pod 是不同的网络命名空间。业务 Pod 访问 **localhost:8126**，不等于能访问同一 Node 上的 Datadog Agent；通常需配置节点地址或挂载共享 Socket。OTel 节点 Collector 也面临类似的寻址问题。

## 2. 为什么不能简单说“Datadog Agent 就是 OTel Collector”？

差别主要出现在收到 Span **之后**。

### Datadog：采集层同时承担 APM 处理

以 **order-service** 的一次请求为例，Datadog Trace Agent 在接收数据后，可能经过以下处理链路（具体策略是否启用，取决于版本和配置）：

~~~mermaid
flowchart LR
    I["收到 Trace"] --> P["规范化 / 过滤<br/>脱敏处理"]
    P --> S["采样"]
    P --> M["APM 统计"]
    S --> T["Trace 上报"]
    M --> U["Stats 上报"]
~~~

**第一，Trace Agent 不是透明转发器。** 它包含规范化字段、过滤标签、混淆敏感信息和采样的能力。它还能将部分采样率反馈给客户端 Tracer，协同控制上报量。

**第二，Trace 和 APM 统计并不是完全相同的数据路径。** 例如请求量、错误率和延迟统计，不应该简单通过最终保留的 Trace 数量计算。Datadog 中，统计可以由客户端 Tracer 预先计算，也可以由 Trace Agent 的 Concentrator 对已接收 Span 聚合；具体方式需要结合版本及配置判断。

这一点解释了 Datadog 为什么把独立采集层做得比较重：它承载了 Datadog 自己的 APM 数据语义，而不仅是减少网络连接。

源码入口：[Datadog Trace Agent](https://github.com/DataDog/datadog-agent/blob/main/pkg/trace/README.md)，可进一步阅读 agent/、sampler/、stats/ 和 writer/。

### OTel：处理能力由 Pipeline 决定

~~~mermaid
flowchart LR
    R["Receivers<br/>接收"] --> P["Processors<br/>处理"]
    P --> E["Exporters<br/>发送"]
~~~

Collector 没有固定的 Datadog 式 APM 处理链。例如，只想将 Span 从 OTLP 转发到 Kafka，可以配置 OTLP Receiver 与 Kafka Exporter；要做批处理、限流、尾部采样，再添加相应组件。

如果启用 **tail_sampling**，多副本 Collector 还要解决同一 Trace 的路由问题。如果启用 **spanmetrics** Connector，可以从 Span 派生指标，但它的指标维度和采样语义不能直接等同于 Datadog APM Stats。

因此，**两者的处理能力有交集，但不是开箱即用的一一对应关系**。参考 [Collector 配置文档](https://opentelemetry.io/docs/collector/configuration/)。

## 3. Datadog Cluster Agent 又是什么？

Datadog Kubernetes 架构里通常还会看到一个 **Cluster Agent**，它作为独立 Deployment 运行，不是每 Node 一个。

~~~mermaid
flowchart TB
    C["Cluster Agent<br/>Deployment"]
    C -. "集群检查 / 元数据 / Admission 等" .-> A["Node Agent A<br/>DaemonSet"]
    C -. "集群级协调" .-> B["Node Agent B<br/>DaemonSet"]
    A --> D["Datadog 后端"]
    B --> D
~~~

Cluster Agent 承担集群检查调度、Admission Controller、外部指标等集群级能力。**普通 Java Trace 不需要经过它才能送到节点上的 Trace Agent。**

这和 Splunk Helm Chart 中独立部署的 **Cluster Receiver** 只有部分相似：两者都把集群级工作放到独立工作负载。但 Cluster Receiver 主要负责采集，Datadog Cluster Agent 还包含平台协调能力，**不能互相当作等价组件**。

参考：[Datadog Cluster Agent 文档](https://docs.datadoghq.com/agent/kubernetes/cluster/)。

## 4. 回到 Spanloom：要学 Datadog 部署 DaemonSet 吗？

**目前没有必要。**

Spanloom 当前的目标是**只采集 Java Span，写入 Kafka，尽量减少独立部署的采集组件**。可以考虑以下三条路径：

~~~mermaid
flowchart TD
    J["spanloom-java-agent"]
    J -->|"直接导出"| K1["Kafka"]
    J -->|"OTLP"| G["Collector Gateway"]
    G --> K2["Kafka"]
    J -->|"OTLP"| N["Collector DaemonSet"]
    N --> K3["Kafka / Gateway"]
~~~

**路径 A：Java Agent → Kafka。** 不部署 Collector，但每个业务 JVM 都要运行 Kafka Producer，并承担 Kafka 连接、认证、缓冲、失败及关闭流程。建议优先评估 [OTel Java Contrib 的 KafkaSpanExporter](https://github.com/open-telemetry/opentelemetry-java-contrib/tree/main/kafka-exporter)，确认 Java 版本、SDK 版本、Agent ClassLoader 隔离和异步失败语义。

**路径 B：Java Agent → Collector Gateway → Kafka。** 只部署集中式 Gateway，JVM 使用现成的 OTLP Exporter；Kafka Client 和凭据留在 Gateway 中。代价是多维护一个独立服务。

**路径 C：Java Agent → Collector DaemonSet → Kafka / Gateway。** 当需要节点指标、容器文件日志、节点本地接入或网络隔离时才有明显价值。仅采集 Span 时，为每个 Node 增加一份 Collector 反而提高部署和运维成本。

从 Datadog 更值得借鉴的是**职责边界**：尽量不要把复杂采样统计、数据治理与后台处理都放进业务 JVM。Spanloom 即使直连 Kafka，也可以让 Kafka 后面的 Ingestion 服务承担这些工作。

## 5. 三个容易混淆的边界

- **Java Agent ≠ Node Agent。** 前者在 JVM 内，后者在 JVM 外，只是名称相似。
- **DaemonSet ≠ Collector 的必经层。** OTel 有 Gateway-only；Spanloom 也可用 Java Contrib Exporter 直连 Kafka。
- **拓扑相似 ≠ 采样及统计行为相同。** Datadog 的 Trace Agent 能力不能直接映射到某个 OTel Processor。

本文是基于公开文档和源码的**架构调研**，未运行 Kubernetes 部署或测量数据。材料于 2026-10-25 查阅；GitHub 链接主要指向会更新的 main 分支，复现实验应固定具体版本。

### 进一步阅读

- [Datadog Helm Chart：Node Agent DaemonSet 和 Cluster Agent](https://github.com/DataDog/helm-charts/blob/main/charts/datadog/README.md)
- [Datadog Trace Agent：接收、处理、采样和 Stats](https://github.com/DataDog/datadog-agent/blob/main/pkg/trace/README.md)
- [OpenTelemetry Collector 部署模式](https://opentelemetry.io/docs/collector/deploy/)
- [本仓库：OTel 官方与 Splunk Collector 部署架构](../otel/collector-deployment.md)
