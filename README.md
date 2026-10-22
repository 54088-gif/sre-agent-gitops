## "K8s SRE智能运维助手"实验手册

------

### **模块一：实验环境构建**

#### **第一部分：基础设施与K8s集群部署 (Ansible)**

创建一个包含1个Master节点和2个Worker节点的K8s集群。

##### **第一步：准备Ansible控制节点**

在管理机器k8s-node1上安装Ansible，并配置SSH免密登录到其他节点。

1. **安装Ansible和sshpass** (sshpass用于在首次连接时传递密码)

   ```shell
   dnf install -y ansible-core sshpass
   ```

2. **配置SSH免密登录** (从k8s-node1到所有节点，包括自己)

   ```shell
   # 生成SSH密钥对
   ssh-keygen -t rsa -N "" -f ~/.ssh/id_rsa
   
   # 将公钥分发到所有节点
   ssh-copy-id -o StrictHostKeyChecking=no root@k8s-node1
   ssh-copy-id -o StrictHostKeyChecking=no root@k8s-node2
   ssh-copy-id -o StrictHostKeyChecking=no root@k8s-node3
   # 期间会提示输入密码，统一输入: native
   ```

   验证免密登录是否成功：ssh root@k8s-node2 "hostname"，如果不需要密码直接返回k8s-node2，则表示成功。

##### **第二步：创建Ansible项目结构**

在k8s-node1的当前目录下（默认为~root），创建sre智能体的项目目录。

```shell
mkdir -p ~/ansible-k8s-deployment
cd ~/ansible-k8s-deployment
```

接下来，在这个目录中创建Ansible所需的文件。

##### **第三步：编写Ansible Playbook文件**

项目采用**Ansible Roles的最佳实践原则**，将不同角色的任务（公共配置、Master配置、Worker配置）分离开，这样会使得Playbook清晰且易于维护。

**1. 创建 ansible.cfg 配置文件**

这个文件为Ansible设置主机清单（inventory）的位置，包括一些基础配置。

~/ansible-k8s-deployment/ansible.cfg

```
[defaults]
inventory      = ./inventory
host_key_checking = False
remote_user    = root
```

**2. 创建 inventory 主机清单文件**

对集群中的所有节点进行功能定义和分组。

~/ansible-k8s-deployment/inventory

```
[masters]
k8s-node1 ansible_host=192.168.30.11

[workers]
k8s-node2 ansible_host=192.168.30.12
k8s-node3 ansible_host=192.168.30.13

[all:vars]
# 在单Master集群需要明确指定Master的IP
# 如果是多Master高可用集群，需要配置VIP
master_ip = 192.168.30.11
# 为K8s Pod网络指定CIDR，Calico网络插件需要
pod_network_cidr = "10.244.0.0/16"
```

**3. 创建 deploy-k8s.yml 主 playbook**

ansible的入口文件，按顺序调用不同的角色。

~/ansible-k8s-deployment/deploy-k8s.yml

```yaml
---
- name: 1. Configure all nodes with common settings
  hosts: all
  become: true
  roles:
    - common

- name: 2. Deploy Kubernetes master node
  hosts: masters
  become: true
  roles:
    - master

- name: 3. Deploy Kubernetes worker nodes
  hosts: workers
  become: true
  roles:
    - worker
    
- name: 4. Label worker nodes from master
  hosts: masters
  become: true
  tasks:
    - name: Apply worker role label to each worker node
      ansible.builtin.command: "kubectl label node {{ item }} node-role.kubernetes.io/worker=worker --overwrite"
      loop: "{{ groups['workers'] }}"    
```

**4. 创建 roles 目录和任务文件**

接下来填充每个角色的具体任务。

```
mkdir -p roles/{common,master,worker}/tasks
```

**(a) 编写 common 角色的任务**
common 角色在所有节点上执行初始化操作。

~/ansible-k8s-deployment/roles/common/tasks/main.yml

```yaml
---
# tasks file for common role
- name: Disable SELinux
  ansible.posix.selinux:
    state: disabled

- name: Disable Swap
  block:
    - name: Turn off swap
      ansible.builtin.command: swapoff -a
    - name: Comment out swap in fstab
      ansible.builtin.replace:
        path: /etc/fstab
        regexp: '^(.*swap.*)$'
        replace: '# \1'
  when: ansible_swaptotal_mb > 0

- name: Configure kernel modules and sysctl for Kubernetes
  block:
    - name: Load overlay module
      community.general.modprobe: { name: overlay, state: present }
    - name: Load br_netfilter module
      community.general.modprobe: { name: br_netfilter, state: present }
    - name: Create k8s.conf for sysctl
      ansible.builtin.copy:
        dest: /etc/sysctl.d/k8s.conf
        content: |
          net.bridge.bridge-nf-call-iptables  = 1
          net.bridge.bridge-nf-call-ip6tables = 1
          net.ipv4.ip_forward                 = 1
    - name: Apply sysctl params
      ansible.builtin.command: sysctl --system

- name: Install and configure containerd
  block:
    - name: Install required packages for containerd
      ansible.builtin.dnf:
        name: [dnf-utils, device-mapper-persistent-data, lvm2]
        state: present
    - name: Add Docker CE repository
      ansible.builtin.get_url:
        url: https://download.docker.com/linux/centos/docker-ce.repo
        dest: /etc/yum.repos.d/docker-ce.repo
    - name: Install containerd.io
      ansible.builtin.dnf:
        name: containerd.io
        state: present
    - name: Configure containerd to use systemd cgroup
      ansible.builtin.shell: |
        mkdir -p /etc/containerd
        containerd config default > /etc/containerd/config.toml
        sed -i 's/SystemdCgroup = false/SystemdCgroup = true/g' /etc/containerd/config.toml
    - name: Restart and enable containerd service
      ansible.builtin.systemd:
        name: containerd
        state: restarted
        enabled: yes

- name: Add Kubernetes v1.32 yum repository
  ansible.builtin.yum_repository:
    name: kubernetes
    description: Kubernetes
    # --- 版本更新点 ---
    baseurl: https://pkgs.k8s.io/core:/stable:/v1.32/rpm/
    gpgkey: https://pkgs.k8s.io/core:/stable:/v1.32/rpm/repodata/repomd.xml.key
    gpgcheck: yes
    enabled: yes
    exclude: kubelet kubeadm kubectl cri-tools kubernetes-cni

- name: Install Kubernetes components
  ansible.builtin.dnf:
    name: [kubelet, kubeadm, kubectl]
    state: present
    disable_excludes: kubernetes

- name: Enable kubelet service
  ansible.builtin.systemd:
    name: kubelet
    enabled: yes
```

**(b) 编写 master 角色的任务**
master 角色的任务只在k8s-node1上执行。

~/ansible-k8s-deployment/roles/master/tasks/main.yml

```yaml
---
# tasks file for master role
- name: Initialize the Kubernetes cluster with kubeadm
  ansible.builtin.shell: "kubeadm init --apiserver-advertise-address={{ master_ip }} --pod-network-cidr={{ pod_network_cidr }} --upload-certs"
  args:
    creates: /etc/kubernetes/admin.conf

- name: Setup kubectl for the root user
  ansible.builtin.file:
    path: /root/.kube
    state: directory
    owner: root
    group: root
    mode: '0755'

- name: Copy admin.conf to user's kube config
  ansible.builtin.copy:
    src: /etc/kubernetes/admin.conf
    dest: /root/.kube/config
    remote_src: yes
    owner: root
    group: root
    mode: '0644'

- name: Install Calico network CNI
  ansible.builtin.shell: "kubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.27.3/manifests/calico.yaml"
  args:
    creates: /etc/cni/net.d/10-calico.conflist

- name: Generate join command
  ansible.builtin.command: kubeadm token create --print-join-command
  register: join_command_raw

- name: Store join command
  ansible.builtin.set_fact:
    join_command: "{{ join_command_raw.stdout }}"
```

**(c) 编写 worker 角色的任务**
 worker 角色只在k8s-node2和k8s-node3上执行。

~/ansible-k8s-deployment/roles/worker/tasks/main.yml

```yaml
---
- name: Join the worker nodes to the cluster
  ansible.builtin.command: "{{ hostvars[groups['masters'][0]]['join_command'] }}"
  args:
    creates: /etc/kubernetes/kubelet.conf
```

*提示：hostvars[groups['masters'][0]]['join_command'] 是Ansible的一个核心技巧，它允许我们从master主机上获取之前步骤中注册的变量（即加入集群的命令），并在worker主机上使用它。*

##### **第四步：执行Ansible Playbook**

```shell
cd ~/ansible-k8s-deployment
ansible-playbook deploy-k8s.yml
```

会发现报错，原因在于代码中调用的ansible.posix 和 community.general的两个内容集合没有安装。 

回到项目根目录 ~/ansible-k8s-deployment：

1. **创建 requirements.yml 文件**

   ~/ansible-k8s-deployment/requirements.yml

   ```yaml
   ---
   collections:
     - name: ansible.posix
       version: ">=1.5.4"  # 指定一个版本范围，保证稳定性
     - name: community.general
       version: ">=8.5.0"
   ```

   *提示：指定版本号可以避免因为集合未来更新可能导致的Playbook不兼容的问题。*

2. **通过文件安装集合**

   使用ansible-galaxy命令下载并安装requirements.yml文件中指定的 ansible.posix 和 community.general 两个集合：

   ```shell
   cd ~/ansible-k8s-deployment
   ansible-galaxy install -r requirements.yml
   ```


重新执行第四步的Ansible Playbook：

```shell
cd ~/ansible-k8s-deployment
ansible-playbook deploy-k8s.yml
```

依据网络的情况，整个过程大概需要5-10分钟，请耐心等待，直到执行完成为止。

##### **第五步：验证集群状态**

当Playbook成功执行完毕后，K8s集群应该已经准备就绪了。在**master节点 (k8s-node1)** 上执行以下命令进行验证：

1. **检查节点状态**

   ```
   kubectl get nodes -o wide
   ```

2. **检查系统Pod状态**

   ```
   kubectl get pods -A
   ```

   确保 kube-system 命名空间下的所有Pods（特别是calico-node, coredns, etcd, kube-apiserver等）都处于 Running 状态。

##### **第六步：核心工具安装**

安装git, tar和helm，为下一步的工作做好准备。

```shell
# 安装git, tar
dnf install -y git tar
# 安装Helm
curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3
chmod 700 get_helm.sh
./get_helm.sh
helm version # 验证
```

------



#### **第二部分：GitOps仓库配置**

1. **在GitHub上创建一个新的、空的、Public的sre-agent-gitops仓库。**

2. **在k8s-node1上克隆并配置仓库**

   ```yaml
   cd ~
   git clone https://github.com/yinling628/sre-agent-gitops.git
   cd sre-agent-gitops
   
   # --- 配置Ollama (YAML方式) ---
   mkdir -p apps/ollama
   cat > apps/ollama/ollama.yaml <<'EOF'
   apiVersion: v1
   kind: Namespace
   metadata:
     name: ai-services
   ---
   apiVersion: apps/v1
   kind: Deployment
   metadata:
     name: ollama
     namespace: ai-services
   spec:
     replicas: 1
     selector:
       matchLabels: { app: ollama }
     template:
       metadata:
         labels: { app: ollama }
       spec:
         containers:
         - name: ollama
           image: ollama/ollama:latest
           ports:
           - containerPort: 11434
   ---
   apiVersion: v1
   kind: Service
   metadata:
     name: ollama-service
     namespace: ai-services
   spec:
     type: NodePort
     selector: { app: ollama }
     ports:
     - port: 11434
       targetPort: 11434
   EOF
   # （此处在旧版中误贴了一个 GitHub token，已删除。请务必到 GitHub 撤销该 token！）
   # 提交并推送
   git add .
   git commit -m "feat: Initial setup with Ollama configuration"
   git push origin main
   ```

------



#### **第三部分：核心组件部署 (Argo CD + Prometheus)**

组件的安装可以使用原生的YAML直接安装，或者使用Helm方式，或者使用Operator方式安装。

1. **安装Argo CD**

   ```shell
   kubectl create namespace argocd
   kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
   
   # kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/v2.13.3/manifests/install.yaml
   
   # 验证: watch kubectl get pods -n argocd
   ```

2. **安装Prometheus**

   ```shell
   # 添加并更新Helm仓库
   helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
   helm repo update
   
   # 使用helm install完成初始安装
   helm install prometheus-stack prometheus-community/kube-prometheus-stack \
   --namespace monitoring \
   --create-namespace \
   --version 57.2.0 \
   --set grafana.service.type=NodePort \
   --set prometheus.service.type=NodePort \
   --set alertmanager.service.type=NodePort
   
   # 验证: watch kubectl get pods -n monitoring (等待所有pods running)
   ```

3. **配置Argo CD并接管全部应用**

   ```yaml
   # 暴露Argo CD UI并获取密码
   kubectl patch svc argocd-server -n argocd -p '{"spec": {"type": "NodePort"}}'
   kubectl get svc argocd-server -n argocd # 查看端口
   kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d; echo # 获取密码
   
   # 在~目录下创建Argo CD Application定义文件
   cd ~
   # ollama-app.yaml
   cat > ollama-app.yaml <<'EOF'
   apiVersion: argoproj.io/v1alpha1
   kind: Application
   metadata:
     name: ollama
     namespace: argocd
   spec:
     project: default
     source:
       repoURL: 'https://github.com/yinling628/sre-agent-gitops.git'
       targetRevision: HEAD
       path: apps/ollama
     destination: { server: 'https://kubernetes.default.svc', namespace: ai-services }
     syncPolicy: { automated: { prune: true, selfHeal: true }, syncOptions: ["CreateNamespace=true"] }
   EOF
   
   # prometheus-app.yaml (接管模式)
   cat > prometheus-app.yaml <<'EOF'
   apiVersion: argoproj.io/v1alpha1
   kind: Application
   metadata:
     name: prometheus-stack
     namespace: argocd
     finalizers:
     - resources-finalizer.argocd.argocd.argoproj.io
   spec:
     project: default
     destination:
       server: 'https://kubernetes.default.svc'
       namespace: monitoring
     syncPolicy:
       automated: { selfHeal: true } # 只开启自愈，不开启自动修剪
     source:
       repoURL: 'https://prometheus-community.github.io/helm-charts'
       chart: kube-prometheus-stack
       targetRevision: 57.2.0
       helm:
         # Argo CD会用这些values来比对，确保安装的状态一致
         values: |
           grafana:
             service:
               type: NodePort
           prometheus:
             service:
               type: NodePort
           alertmanager:
             service:
               type: NodePort
   EOF
   
   # 应用
   kubectl apply -f ollama-app.yaml
   kubectl apply -f prometheus-app.yaml
   ```
   
   在Argo CD图形界面中，如果 ollama和 prometheus-stack 的状态都是绿色，表明所有核心组件均通过最佳实践部署并稳定运行。
   
4. **登录 Argo，Prometheus 和 Grafana**

   ```shell
   # 登录 Argo CD
   kubectl get svc argocd-server -n argocd 
   kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d; echo
   https://192.168.30.11:31397  # 使用你自己的端口
    
   # 登录Prometheus和Grafana
   kubectl get svc -n monitoring
   http://192.168.30.11:30090 
   http://192.168.30.11:31358  # Grafana的默认用户名/密码是 admin/prom-operator
   ```

截至现在，**K8s 1.32, Argo, Prometheus, Grafana, Ollama 这些基础设施均已经就绪**，接着就要进入SRE Agent开发阶段。

在继续下一模块之前，关闭三台虚拟机，分别拍摄快照，以备在后续项目故障时进行快速恢复，或者满足重复实验的需要。

------

#### **模块一重点关注的内容：**

1. Ansible，除了ansible.builtin外，如何安装和使用非Ansible自带的内容集合，比如ansible.posix 和 community.general 等。
2. GitOps，要熟练使用。在简历加入自己的GitHub账户，比博客更有吸引力。
3. Argo CD，练习中要理解CICD的原理，对使用Argo CD管理项目的流程要有自己的见解。
4. Helm，要能够解释Helm在云原生软件的安装和管理的优缺点。
5. Prometheus，解释Prometheus的作用。 # 可以尝试使用operator安装Prometheus。
6. Grafana，可视化系统监控的配置与使用。
7. 在完成模块一的过程中都遇到了哪些问题，最后是如何解决的。
8. 能准确描述作为AI智能体基础架构的云原生环境的完整结构，以及各个组件的作用。

------



### **模块二：智能体开发**

#### **1. 目标**

开发一个基于Go语言的AI智能体，使其能够：

- **感知** Prometheus中的PodCrashLooping告警。
- **思考** 通过调用Ollama LLM服务，根据告警信息决策是否需要重启Pod。
- **行动** 调用Kubernetes API执行重启操作（通过删除Pod实现）。

#### **2. 开发环境准备 (k8s-node1)**

1. **安装Go语言**:

   ```shell
   dnf install -y golang
   go version
   ```

2. **创建项目并初始化模块**:

   ```shell
   mkdir -p ~/sre-agent
   cd ~/sre-agent
   go mod init sre-agent
   ```

3. **安装所有Go依赖**:
   
   ```shell
   # 在sre-agent目录下先创建一个空的 main.go，然后运行 tidy
   touch main.go 
   # (此时main.go内容为空，但足以让tidy工作)
   
   # 手动添加几个核心依赖，以确保版本
   go get k8s.io/client-go@v0.30.0
   go get github.com/prometheus/common@v0.53.0
   
   # 运行tidy来处理所有间接依赖
   go mod tidy
   ```
   
   *（注意：更简单的做法是写完代码后再运行go mod tidy一次，它会自动处理所有import的包）*

#### **3. 准备故障与监控环境**

1. **部署持续崩溃的应用 (crash-app)**:

   ```yaml
   cat > ~/crash-app.yaml <<EOF
   apiVersion: apps/v1
   kind: Deployment
   metadata:
     name: crash-app
   spec:
     replicas: 1
     selector:
       matchLabels:
         app: crash-app
     template:
       metadata:
         labels:
           app: crash-app
       spec:
         containers:
         - name: main
           image: alpine:latest
           command: ["/bin/sh", "-c", "echo 'I am designed to fail!'; exit 1"]
   EOF
   
   kubectl apply -f ~/crash-app.yaml
   # 验证: watch kubectl get pods (会看到crash-app处于CrashLoopBackOff状态)
   ```

2. **创建Prometheus告警规则**:

   ```yaml
   cat > ~/pod-crash-rule.yaml <<EOF
   apiVersion: monitoring.coreos.com/v1
   kind: PrometheusRule
   metadata:
     name: pod-crash-alert
     namespace: monitoring
     labels:
       release: prometheus-stack # 确保operator能发现此规则
   spec:
     groups:
     - name: kubernetes.rules
       rules:
       - alert: PodCrashLooping
         expr: kube_pod_container_status_waiting_reason{reason="CrashLoopBackOff"} == 1
         for: 1m
         labels:
           severity: critical
         annotations:
           summary: "Pod is crash looping"
           description: "Pod {{ \$labels.namespace }}/{{ \$labels.pod }} is in CrashLoopBackOff state."
   EOF
   
   kubectl apply -f ~/pod-crash-rule.yaml
   # 验证: 在Prometheus UI的Alerts页面，等待看到PodCrashLooping告警变为FIRING状态。
   ```

3. **确保Ollama模型已拉取**:

   ```shell
   # 找到Ollama Pod名
   OLLAMA_POD=$(kubectl get pods -n ai-services -l app=ollama -o jsonpath='{.items[0].metadata.name}')
   # 进入Pod拉取模型
   kubectl exec -it $OLLAMA_POD -n ai-services -- ollama pull qwen2.5:1.5b
   # 验证
   kubectl exec -it $OLLAMA_POD -n ai-services -- ollama list | grep 'qwen2.5:1.5b'
   ```

​       选择qwen2.5:1.5b的根本原因是笔记本电脑的配置限制。

#### **4. Go实现的Agent代码 (~/sre-agent/main.go)**

```go
package main

import (
	"bytes"
	"context"
	"encoding/json"
	"fmt"
	"io"
	"log"
	"net/http"
	"os"
	"path/filepath"
	"strings"
	"time"

	"k8s.io/client-go/kubernetes"
	"k8s.io/client-go/tools/clientcmd"

	v1 "k8s.io/api/core/v1"
	metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
)

// --- 配置常量 ---
const (
	prometheusURL = "http://192.168.30.11:30090" // 替换成你的Prometheus NodePort IP和端口
	ollamaURL     = "http://192.168.30.11:31982" // 替换成你的Ollama NodePort IP和端口 
	ollamaModel   = "qwen2.5:1.5b"
)

// --- 数据结构定义 ---

// Prometheus告警响应结构体
type PrometheusAlerts struct {
	Status string `json:"status"`
	Data   struct {
		Alerts []Alert `json:"alerts"`
	} `json:"data"`
}

// 单个告警的结构体
type Alert struct {
	Labels      map[string]string `json:"labels"`
	Annotations map[string]string `json:"annotations"`
	State       string            `json:"state"`
	ActiveAt    time.Time         `json:"activeAt"`
}

// Ollama请求结构体
type OllamaRequest struct {
	Model  string `json:"model"`
	Prompt string `json:"prompt"`
	Stream bool   `json:"stream"`
}

// Ollama响应结构体
type OllamaResponse struct {
	Response string `json:"response"`
}

// --- 主函数 ---
func main() {
	log.Println("Starting K8s SRE Agent...")

	// 初始化Kubernetes客户端
	clientset, err := getKubernetesClient()
	if err != nil {
		log.Fatalf("Error creating Kubernetes client: %v", err)
	}

	// 启动智能体的主循环
	agentLoop(clientset)
}

// 智能体主循环
func agentLoop(clientset *kubernetes.Clientset) {
	for {
		log.Println("--- Agent Loop Start ---")

		// 1. 感知：查询Prometheus告警
		alerts, err := queryPrometheusAlerts()
		if err != nil {
			log.Printf("Error querying Prometheus: %v", err)
			time.Sleep(1 * time.Minute) // 如果查询失败，等待更长时间
			continue
		}

		if len(alerts) == 0 {
			log.Println("No active alerts. System is healthy.")
		} else {
			log.Printf("Found %d active alert(s).", len(alerts))
			// 2. 思考与行动：处理每个告警
			for _, alert := range alerts {
				if alert.Labels["alertname"] == "PodCrashLooping" {
					handlePodCrashLooping(clientset, alert)
				}
			}
		}

		log.Println("--- Agent Loop End ---")
		time.Sleep(30 * time.Second) // 每30秒检查一次
	}
}

// 感知 (Perception) 
func queryPrometheusAlerts() ([]Alert, error) {
	resp, err := http.Get(prometheusURL + "/api/v1/alerts")
	if err != nil {
		return nil, fmt.Errorf("failed to get alerts from Prometheus: %w", err)
	}
	defer resp.Body.Close()

	if resp.StatusCode != http.StatusOK {
		return nil, fmt.Errorf("prometheus returned non-200 status code: %d", resp.StatusCode)
	}

	body, err := io.ReadAll(resp.Body)
	if err != nil {
		return nil, fmt.Errorf("failed to read response body: %w", err)
	}

	var alertResponse PrometheusAlerts
	if err := json.Unmarshal(body, &alertResponse); err != nil {
		return nil, fmt.Errorf("failed to unmarshal json: %w", err)
	}

	var firingAlerts []Alert
	for _, alert := range alertResponse.Data.Alerts {
		if alert.State == "firing" {
			firingAlerts = append(firingAlerts, alert)
		}
	}
	return firingAlerts, nil
}

func handlePodCrashLooping(clientset *kubernetes.Clientset, alert Alert) {
    podName := alert.Labels["pod"]
    namespace := alert.Labels["namespace"]
    reason := alert.Annotations["description"] // 从告警的描述中获取原因
    log.Printf("Handling PodCrashLooping alert for %s/%s", namespace, podName)

    // 不再需要收集详细上下文，直接请求LLM决策
    decision, err := getLLMDecision(namespace, podName, reason)
    if err != nil {
        log.Printf("Failed to get decision from LLM: %v", err)
        return
    }
    log.Printf("LLM Decision: %s", decision)

    // 执行行动
    if err := executeAction(clientset, namespace, podName, decision); err != nil {
        log.Printf("Failed to execute action '%s' for pod %s/%s: %v", decision, namespace, podName, err)
    }
}

// 收集Pod的详细信息
func getPodContext(clientset *kubernetes.Clientset, namespace, podName string) (string, error) {
	var contextBuilder strings.Builder

	// 获取Pod对象
	pod, err := clientset.CoreV1().Pods(namespace).Get(context.TODO(), podName, metav1.GetOptions{})
	if err != nil {
		return "", err
	}
	contextBuilder.WriteString(fmt.Sprintf("Pod Status: %s\n", pod.Status.Phase))
	if len(pod.Status.ContainerStatuses) > 0 {
		status := pod.Status.ContainerStatuses[0]
		if status.State.Waiting != nil {
			contextBuilder.WriteString(fmt.Sprintf("Container Waiting Reason: %s\n", status.State.Waiting.Reason))
		}
	}

	// 获取Pod日志
	podLogOpts := v1.PodLogOptions{TailLines: new(int64)}
	*podLogOpts.TailLines = 50 // 获取最后50行日志
	req := clientset.CoreV1().Pods(namespace).GetLogs(podName, &podLogOpts)
	podLogs, err := req.Stream(context.TODO())
	if err != nil {
		return "", fmt.Errorf("error in opening stream: %w", err)
	}
	defer podLogs.Close()
	buf := new(bytes.Buffer)
	_, err = io.Copy(buf, podLogs)
	if err != nil {
		return "", fmt.Errorf("error in copy information from podLogs to buf: %w", err)
	}
	contextBuilder.WriteString(fmt.Sprintf("\n--- Pod Logs ---\n%s\n", buf.String()))

	// 获取Pod事件
	events, err := clientset.CoreV1().Events(namespace).List(context.TODO(), metav1.ListOptions{
		FieldSelector: "involvedObject.name=" + podName,
	})
	if err != nil {
		return "", err
	}
	contextBuilder.WriteString("\n--- Pod Events ---\n")
	for _, event := range events.Items {
		contextBuilder.WriteString(fmt.Sprintf("- %s: %s\n", event.Reason, event.Message))
	}

	return contextBuilder.String(), nil
}

// 请求LLM进行分析和决策
func getLLMDecision(namespace, podName, reason string) (string, error) {
    prompt := fmt.Sprintf(
        `A pod is in a CrashLoopBackOff state.
        - Namespace: %s
        - Pod: %s
        - Reason: %s
        Based on this, should I restart the pod? Your response should contain the word "restart" if you recommend restarting.`, namespace, podName, reason)

    reqBody, err := json.Marshal(OllamaRequest{
        Model:  ollamaModel,
        Prompt: prompt,
        Stream: false,
    })
    if err != nil {
        return "", err
    }

    resp, err := http.Post(ollamaURL+"/api/generate", "application/json", bytes.NewBuffer(reqBody))
    if err != nil {
        return "", err
    }
    defer resp.Body.Close()

    body, err := io.ReadAll(resp.Body)
    if err != nil {
        return "", err
    }

    var ollamaResp OllamaResponse
    if err := json.Unmarshal(body, &ollamaResp); err != nil {
        log.Printf("DEBUG: Failed to unmarshal LLM response body: %s", string(body))
        return "", err
    }

    responseText := strings.ToLower(ollamaResp.Response) // 转换为小写
    log.Printf("DEBUG: Raw LLM response: '%s'", ollamaResp.Response)

    if strings.Contains(responseText, "restart") {
        log.Println("LLM response contains 'restart'. Interpreting as RESTART action.")
        return "RESTART", nil
    }
    
    log.Println("LLM response does not contain 'restart'. Interpreting as IGNORE action.")
    return "IGNORE", nil
    // ---------------------
}

// 执行恢复动作
func executeAction(clientset *kubernetes.Clientset, namespace, podName, action string) error {
	log.Printf("Executing action: %s on pod %s/%s", action, namespace, podName)
	switch action {
	case "RESTART":
		// 在K8s中，重启一个由Deployment管理的Pod最简单的方法是删除它
		// Deployment Controller会自动创建一个新的Pod来替代
		log.Printf("Deleting pod %s/%s to trigger restart...", namespace, podName)
		return clientset.CoreV1().Pods(namespace).Delete(context.TODO(), podName, metav1.DeleteOptions{})
	case "IGNORE":
		log.Println("Action is IGNORE. No operation will be performed.")
		return nil
	default:
		return fmt.Errorf("unknown action: %s", action)
	}
}

// 获取K8s客户端
func getKubernetesClient() (*kubernetes.Clientset, error) {
	kubeconfig := filepath.Join(os.Getenv("HOME"), ".kube", "config")
	config, err := clientcmd.BuildConfigFromFlags("", kubeconfig)
	if err != nil {
		return nil, fmt.Errorf("error building kubeconfig: %w", err)
	}
	// }
	clientset, err := kubernetes.NewForConfig(config)
	if err != nil {
		return nil, fmt.Errorf("error creating clientset: %w", err)
	}
	return clientset, nil
}
```

#### **5. 运行与验证**

1. **打开两个终端**。

2. **终端一**: 运行Agent

   ```shell
   cd ~/sre-agent
   go run main.go
   ```

3. **终端二**: 监控Pods

   ```shell
   watch kubectl get pods
   ```

4. **验证**: 在终端一看到Deleting pod ...的日志后，切换到终端二，观察到crash-app的Pod被Terminating并被一个新的Pod替换。

------

#### **模块二重点关注的内容：**

1. Golang，安装过程、目录结构和程序结构。对main.go程序流程能讲清楚。
2. Ollama，能讲清楚Ollama和大模型的使用。Qwen大模型在agent中的作用。为什么选择qwen2.5:1.5b。
3. 在完成模块二的过程中所遇到的问题，最后是如何解决的。

------



### **模块三：容器化与GitOps部署**

#### **1. 目标**

将模块二开发的Go语言SRE Agent，通过容器化和GitOps流程，作为一个稳定、可靠的云原生应用部署到我们的Kubernetes集群中。

#### **2. 准备工作**

- **确认代码**: 模块二最终版的main.go已在~/sre-agent目录中准备就绪。
- **确认环境**: Kubernetes v1.32集群及所有核心组件（Argo CD, Prometheus, Ollama）均健康运行。

#### **3. 容器化 Agent**

- **安装rest包**:

  ```shell
  cd ~/sre-agent
  go get k8s.io/client-go/rest
  go mod tidy
  ```

- **修改main.go中的getKubernetesClient函数**

  让代码能自动识别并使用Kubernetes提供的“in-cluster”配置，使其优先尝试in-cluster配置。

  ```go
  import (
      // ...其他import
      "k8s.io/client-go/rest" // 确保导入
  )
  
  // ...
  
  // 辅助: 获取K8s客户端
  func getKubernetesClient() (*kubernetes.Clientset, error) {
      config, err := rest.InClusterConfig() // 优先尝试In-Cluster配置
      if err != nil {
          log.Printf("Failed to get in-cluster config: %v. Falling back to kubeconfig for local dev.", err)
          // 回退到使用kubeconfig文件
          homeDir := os.Getenv("HOME")
          if homeDir == "" {
               return nil, fmt.Errorf("HOME environment variable not set")
          }
          kubeconfig := filepath.Join(homeDir, ".kube", "config")
          config, err = clientcmd.BuildConfigFromFlags("", kubeconfig)
          if err != nil {
              return nil, fmt.Errorf("error building kubeconfig: %w", err)
          }
      }
  
      clientset, err := kubernetes.NewForConfig(config)
      if err != nil {
          return nil, fmt.Errorf("error creating clientset: %w", err)
      }
      return clientset, nil
  }
  ```

- **在~/sre-agent目录下创建Dockerfile文件**:

  ```dockerfile
  # --- Stage 1: Build ---
  # 使用官方的Go镜像作为构建环境
  FROM golang:alpine AS builder
  
  # 设置工作目录
  WORKDIR /app
  
  # 复制go.mod和go.sum文件，并下载依赖
  # 这一步可以被Docker缓存，加快后续构建速度
  COPY go.mod go.sum ./
  RUN go mod download
  
  # 复制所有源代码
  COPY . .
  
  # 编译Go程序。
  # CGO_ENABLED=0: 禁用CGO，构建一个纯Go的静态二进制文件。
  # -o /sre-agent: 将编译好的程序输出到根目录下的sre-agent文件。
  RUN CGO_ENABLED=0 GOOS=linux go build -o /sre-agent .
  
  # --- Stage 2: Final Image ---
  # 使用一个极小的基础镜像alpine
  FROM alpine:latest
  
  # 从构建阶段(builder)复制编译好的二进制文件到当前镜像
  COPY --from=builder /sre-agent /sre-agent
  
  # 设置容器的入口点。当容器启动时，它会执行/sre-agent这个程序。
  ENTRYPOINT ["/sre-agent"]
  ```

- **在所有节点安装Docker和ctr工具**:

  解决实验环境下没有中央镜像仓库的问题，并处理docker与containerd镜像存储隔离的问题。

  ```yaml
  vim install-docker.yml
  
  ---
  - name: Install Docker Engine on all nodes
    hosts: all  # 这个play将会在inventory中的所有主机上运行
    become: true # 使用sudo权限执行任务
    tasks:
      - name: Ensure required packages for adding repo are installed
        # 使用dnf模块确保dnf-utils存在
        ansible.builtin.dnf:
          name: dnf-utils
          state: present
  
      - name: Add Docker CE repository
        # 使用get_url模块下载repo文件，这是幂等的
        ansible.builtin.get_url:
          url: https://download.docker.com/linux/centos/docker-ce.repo
          dest: /etc/yum.repos.d/docker-ce.repo
          mode: '0644'
  
      - name: Install Docker packages
        # 使用dnf模块安装docker-ce, docker-ce-cli, 和 containerd.io
        # 我们也把containerd.io加进来，确保所有CRI工具都通过包管理器安装
        # 加上containerd-ctr是为了确保ctr命令一定存在
        ansible.builtin.dnf:
          name:
            - docker-ce
            - docker-ce-cli
            - containerd.io
          state: present
          update_cache: yes # 安装前先更新dnf缓存
  
      - name: Start and enable Docker service
        # 使用systemd模块来管理服务
        ansible.builtin.systemd:
          name: docker
          state: started
          enabled: yes
  ```

- **构建Docker镜像并导入到Containerd**

  ```shell
  cd ~/sre-agent
  
  # 1. 使用Docker构建
  docker build -t sre-agent:v1.0 .
  
  # 2. 将Docker镜像保存为tar包
  docker save sre-agent:v1.0 -o sre-agent-v1.0.tar
  
  # 3. 使用ctr导入到containerd的k8s.io命名空间
  ctr -n k8s.io image import sre-agent-v1.0.tar
  
  # 4. (可选) 验证导入成功
  ctr -n k8s.io image ls | grep sre-agent
  ```

- **将sre-agent-v1.0镜像导入到另外两个worker节点：**

  ```shell
  scp sre-agent-v1.0.tar root@k8s-node2:/root
  scp sre-agent-v1.0.tar root@k8s-node3:/root
  ctr -n k8s.io image import sre-agent-v1.0.tar
  
  # kubectl rollout restart deployment/sre-agent -n default # 如果有重构镜像，执行
  ```

#### **4. GitOps部署**

声明式地定义Agent的运行方式及其所需权限，让Argo CD自动化地部署和管理Agent。

- **创建目录**

  ```shell
  cd ~/sre-agent-gitops
  mkdir -p apps/sre-agent
  ```

- **创建Deployment配置 (apps/sre-agent/deployment.yaml)**

  ```yaml
  apiVersion: apps/v1
  kind: Deployment
  metadata:
    name: sre-agent
    namespace: default
  spec:
    replicas: 1
    selector:
      matchLabels:
        app: sre-agent
    template:
      metadata:
        labels:
          app: sre-agent
      spec:
        serviceAccountName: sre-agent-sa # 关联ServiceAccount
        containers:
        - name: agent
          image: sre-agent:v1.0 # 使用我们本地构建的镜像
          imagePullPolicy: IfNotPresent # 关键！强制使用本地镜像
  ```
  
- **创建RBAC配置 (apps/sre-agent/rbac.yaml)** 

  ```yaml
  apiVersion: v1
  kind: ServiceAccount
  metadata:
    name: sre-agent-sa
    namespace: default # Agent将部署在default命名空间
  
  ---
  apiVersion: rbac.authorization.k8s.io/v1
  kind: ClusterRole
  metadata:
    name: sre-agent-role
  rules:
  - apiGroups: [""] # 核心API组
    resources: ["pods", "pods/log", "events"]
    verbs: ["get", "list", "watch"]
  - apiGroups: [""]
    resources: ["pods"]
    verbs: ["delete"]
  
  ---
  apiVersion: rbac.authorization.k8s.io/v1
  kind: ClusterRoleBinding
  metadata:
    name: sre-agent-binding
  subjects:
  - kind: ServiceAccount
    name: sre-agent-sa
    namespace: default
  roleRef:
    kind: ClusterRole
    name: sre-agent-role
    apiGroup: rbac.authorization.k8s.io
  ```

- **提交并推送到Git**

  ```shell
  git add apps/sre-agent/
  git commit -m "feat: Add Kubernetes manifests for SRE Agent"
  git push origin main
  ```

- **在k8s-node1的~root目录下创建sre-agent-app.yaml并应用**

  ```yaml
  apiVersion: argoproj.io/v1alpha1
  kind: Application
  metadata:
    name: sre-agent
    namespace: argocd
  spec:
    project: default
    source:
      repoURL: 'https://github.com/yinling628/sre-agent-gitops.git' # 你的仓库URL
      targetRevision: HEAD
      path: apps/sre-agent
    destination:
      server: 'https://kubernetes.default.svc'
      namespace: default # 部署到default命名空间
    syncPolicy:
      automated:
        prune: true
        selfHeal: true
  ```

#### **5. 最终验证**

1. **Argo CD UI**: 确认sre-agent应用状态为Healthy和Synced。

2. **命令行**:

   ```shell
   # 确认Agent Pod正在运行
   kubectl get pods -l app=sre-agent
   # 获取Pod全名
   AGENT_POD=$(kubectl get pods -l app=sre-agent -o jsonpath='{.items[0].metadata.name}')
   # 查看Agent的实时日志，确认它在正常工作
   kubectl logs -f $AGENT_POD
   
   # 打开另一个终端，观察crash-app是否被自动重启
   watch kubectl get pods
   ```

------

#### **模块三重点关注的内容：**

1. 在集群重启后，会发生什么状况？请分析原因并给出最终的解决方案。
2. 在完成模块三的过程中遇到了哪些令你难忘的问题，你是如何解决的。
3. 谈谈你对整个项目的处理流程的认识。

------

