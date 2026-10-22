# Prometheus 告警规则

本目录存放 Prometheus 告警规则（`PrometheusRule`）。

- `pod-crash-rule.yaml`：定义 `PodCrashLooping` 告警，是 SRE Agent「感知」环节的触发源。

> 该规则通过 `release: prometheus-stack` 标签被 Prometheus Operator 发现，
> 依赖模块一中通过 Helm 安装的 `kube-prometheus-stack`（CRD：`monitoring.coreos.com/v1`）。
