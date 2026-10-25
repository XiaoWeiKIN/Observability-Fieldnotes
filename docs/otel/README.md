# OpenTelemetry Notes

## Topics

### Specification and architecture
- API vs SDK
- Resource
- InstrumentationScope
- Context and Baggage
- Propagators
- Semantic Conventions
- OTLP

### Signals
- Traces
- Metrics
- Logs
- Profiles

### Instrumentation
- Manual instrumentation
- Auto-instrumentation
- Zero-code instrumentation
- eBPF-based instrumentation
- Kubernetes Operator injection

### Collector

- [Collector 部署架构：OTel 官方模式与 Splunk 实践](collector-deployment.md) — Agent / Gateway / Agent-to-Gateway、Kubernetes、扩容与 Kafka

```mermaid
flowchart LR
    R[Receivers] --> P[Processors]
    P --> C[Connectors]
    C --> E[Exporters]
    P --> E
    X[Extensions] -. support .-> R
    X -. support .-> P
    X -. support .-> E
```

## Questions to answer

1. Which semantic conventions are stable vs development?
2. What compatibility contracts does OTLP provide?
3. How do SDK batching and Collector batching interact?
4. How does tail sampling affect trace completeness and memory?
5. What telemetry should never leave the workload boundary?
6. How do Collector failure modes surface back to applications?
