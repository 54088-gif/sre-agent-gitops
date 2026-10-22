# Argo CD 使用指南

本目录存放整个项目的 Argo CD `Application` 清单。Argo CD 是 GitOps 的核心：它以
**Git 仓库为唯一事实来源（Single Source of Truth）**，持续监控仓库变化，并把声明的
目标状态自动同步到 Kubernetes 集群。

> 本文档覆盖 Argo CD 的安装、访问、核心概念、本仓库 4 个 Application 的使用方法，
> 以及同步策略与常见排错。配套的坑点总结见项目根目录 `注意事项.md`。

---

## 1. GitOps 与 Argo CD 理念

传统 CD 是「推送式」：CI 流水线把构建产物推送到集群。GitOps 则是「拉取式」：

```
开发者 ──git push──> Git 仓库 ──Argo CD 持续监控──> 自动同步 ──> K8s 集群
```

- **声明式**：所有期望状态都以 YAML 存于 Git，可审计、可回滚。
- **自愈（selfHeal）**：有人手动改了集群，Argo CD 会把它拉回 Git 声明值。
- **无 Drift**：集群实际状态与 Git 不一致时，UI 会显示 `OutOfSync`。

---

## 2. 安装 Argo CD

```shell
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# 若需固定版本，改用具体 tag，例如 v2.13.3：
# kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/v2.13.3/manifests/install.yaml

# 验证（等待所有 Pod Running）
watch kubectl get pods -n argocd
```

安装后 `argocd` 命名空间下会出现：`argocd-server`（Web UI / API）、`argocd-repo-server`
（拉取 Git 仓库）、`argocd-application-controller`（执行同步）等核心组件。

---

## 3. 访问 Argo CD

### 3.1 暴露 Web UI

实验环境无 LoadBalancer，用 NodePort 暴露：

```shell
kubectl patch svc argocd-server -n argocd -p '{"spec": {"type": "NodePort"}}'
kubectl get svc argocd-server -n argocd        # 查看分配的 NodePort 端口
```

浏览器访问 `https://<节点IP>:<NodePort>`（例如 `https://192.168.30.11:31397`）。
首次访问会提示证书不受信任，跳过即可。

### 3.2 获取初始密码

```shell
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath="{.data.password}" | base64 -d; echo
```

默认用户名 `admin`，密码即上面输出的内容。登录后建议立即修改密码：

```shell
argocd account update-password   # 需要先安装 argocd CLI 并登录
```

### 3.3 安装 CLI（可选）

```shell
curl -sSL -o argocd-linux-amd64 https://github.com/argoproj/argo-cd/releases/latest/download/argocd-linux-amd64
install -m 555 argocd-linux-amd64 /usr/local/bin/argocd

argocd login <NodePort地址> --username admin --password <密码> --insecure
```

---

## 4. 核心概念

| 概念 | 说明 |
| --- | --- |
| **Application** | Argo CD 的核心资源，描述「从哪个仓库的哪个目录，同步到哪个集群的哪个命名空间」 |
| **Project** | Application 的逻辑分组，内置 `default` 项目即可满足本实验 |
| **Source** | 状态来源：Git 目录（`path`）或 Helm Chart（`chart`） |
| **Destination** | 目标：集群 API 地址 + 命名空间 |
| **Sync（同步）** | 把 Source 声明的状态应用到 Destination 的过程 |
| **Healthy / Degraded** | 应用的健康状态（如 Deployment 副本是否就绪） |
| **Synced / OutOfSync** | 集群是否与 Git 声明一致 |

### Application 关键字段对照

以 `sre-agent-app.yaml` 为例：

```yaml
spec:
  project: default                      # 所属项目
  source:
    repoURL: 'https://github.com/<你的账号>/sre-agent-gitops.git'
    targetRevision: HEAD                # 跟踪分支（HEAD=默认分支，也可写 main/tag/commit）
    path: apps/sre-agent                # 清单所在目录
  destination:
    server: 'https://kubernetes.default.svc'   # 集群内 API 地址
    namespace: default                  # 目标命名空间
  syncPolicy:
    automated:
      prune: true                       # 自动删除 Git 中已移除的资源
      selfHeal: true                    # 自动纠正偏离 Git 的手动改动
```

---

## 5. 本仓库的 4 个 Application

| 文件 | 应用 | 目标命名空间 | 类型 | 说明 |
| --- | --- | --- | --- | --- |
| `ollama-app.yaml` | ollama | ai-services | Git 目录 | AI 服务（决策大脑） |
| `sre-agent-app.yaml` | sre-agent | default | Git 目录 | Go SRE Agent（执行中枢） |
| `crash-app-app.yaml` | crash-app | default | Git 目录 | 模拟故障源 |
| `prometheus-app.yaml` | prometheus-stack | monitoring | Helm Chart 接管 | 管理 Prometheus 栈 |

### 5.1 Git 目录型（ollama / sre-agent / crash-app）

以 `ollama-app.yaml` 为例，还额外使用了 `syncOptions`：

```yaml
syncPolicy:
  automated:
    prune: true
    selfHeal: true
  syncOptions:
  - CreateNamespace=true    # 目标命名空间不存在时自动创建（ai-services）
```

### 5.2 Helm 接管型（prometheus-app.yaml）⭐

`prometheus-app.yaml` 与前三个不同——它**不**指向 Git 目录，而是直接引用 Helm Chart 仓库，
用于「接管」模块一中用 `helm install` 装好的 Prometheus 栈，让它纳入 Argo CD 管理：

```yaml
metadata:
  finalizers:
  - resources-finalizer.argocd.argoproj.io   # 删除 Application 时级联清理资源
spec:
  syncPolicy:
    automated:
      selfHeal: true    # 注意：只开自愈，不开自动修剪
  source:
    repoURL: 'https://prometheus-community.github.io/helm-charts'
    chart: kube-prometheus-stack
    targetRevision: 57.2.0
    helm:
      values: |
        grafana: { service: { type: NodePort } }
        prometheus: { service: { type: NodePort } }
        alertmanager: { service: { type: NodePort } }
```

**接管模式的三条铁律**（详见 `注意事项.md`）：

1. 必须加 `resources-finalizer` finalizer，否则删除 Application 时资源会残留；
2. 只 `selfHeal`、**不 `prune`**——Helm 会渲染大量资源，自动修剪有误删风险；
3. `helm.values` 必须与最初 `helm install --set ...` 完全一致，否则会一直 `OutOfSync` 反复 diff。

---

## 6. 部署 Application

替换好 `repoURL` 后，依次应用：

```shell
kubectl apply -f argocd/ollama-app.yaml
kubectl apply -f argocd/crash-app-app.yaml
kubectl apply -f argocd/sre-agent-app.yaml
kubectl apply -f argocd/prometheus-app.yaml
```

> 也可以只应用一个「入口」Application，让它去创建其余 Application（App-of-Apps 模式），
> 本实验规模较小，逐个 apply 即可。

---

## 7. 验证与排错

### 7.1 UI 验证

在 Argo CD Web UI 中，每个 Application 应显示为 **绿色**（`Healthy` + `Synced`）。
- 黄色 `OutOfSync` → 集群与 Git 不一致，点击 `Sync` 或检查 selfHeal 是否开启；
- 红色 `Degraded` → 资源健康检查失败，点进应用查看 Pod 状态。

### 7.2 命令行验证

```shell
kubectl get applications -n argocd          # 查看所有 Application
kubectl get pods -l app=sre-agent           # 确认 Agent 已运行
kubectl get pods -n ai-services -l app=ollama
```

### 7.3 常见问题

| 现象 | 可能原因 | 处理 |
| --- | --- | --- |
| 一直 `Unknown` / 无法拉取仓库 | `repoURL` 写错或仓库私有未配凭据 | 检查 URL，私有仓库需配 `Repositories` 凭据 |
| `OutOfSync` 且反复出现 | selfHeal 未开，或 values 不一致 | 开启 selfHeal；核对 Helm values |
| sre-agent Pod `ImagePullBackOff` | 本地镜像未导入到该节点 | 在每个节点 `ctr -n k8s.io image import` |
| Pod 起不来 | RBAC 权限不足 | 核对 `apps/sre-agent/rbac.yaml` 是否已同步 |
| UI 登录不上 | NodePort 未暴露 / 密码错误 | 重做 3.1 / 3.2 |

---

## 8. 常用 CLI 命令速查

```shell
argocd app list                        # 列出应用及同步/健康状态
argocd app get <app>                   # 查看应用详情
argocd app sync <app>                  # 手动触发同步
argocd app diff <app>                  # 对比 Git 与集群差异
argocd app history <app>               # 查看同步历史（便于回滚）
argocd app rollback <app> <revision>   # 回滚到某次同步
argocd login <host> --insecure         # 登录
```

---

## 9. 相关文档

- 实验手册（三大模块）：`../README.md`
- 坑点与注意事项：项目根目录 `../../注意事项.md`
- Argo CD 官方文档：https://argo-cd.readthedocs.io/
