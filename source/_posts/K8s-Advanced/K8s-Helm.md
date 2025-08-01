---
title: K8s | Helm
description: 'Kubernetes 套件管理工具 Helm 的介紹、安裝與使用教學，並比較 Helm 2 與 Helm 3 的差異'
date: 2025-07-30 23:17:33
categories: Kubernetes
tags:
- Helm
- Package Management
- Chart
---

# What is Helm?

Helm 是 Kubernetes 的「套件管理工具」，可協助你：
- 安裝複雜應用程式（如 WordPress、MySQL、Prometheus）
- 管理部署版本與參數設定
- 執行升級（upgrade）、回滾（rollback）、移除（uninstall）等操作

Helm 將應用打包成一個「Chart」，其中包含：
- Kubernetes YAML 模板（template）
- 預設變數設定檔 `values.yaml`
- 說明檔與 Chart metadata（如 `Chart.yaml`）


## Basic Helm Commands
> [Artifact HUB](https://artifacthub.io/) is a web-based application that enables finding, installing, and publishing Cloud Native packages.

範例：安裝 WordPress
```bash
# 新增套件來源並更新
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update

# 安裝應用
helm install wordpress bitnami/wordpress
helm install <release-name> <chart>
```

### 其他常用命令

#### Chart Repository 操作
```bash
helm repo list                         # 查看已加入的 repository
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update                       # 更新 chart repository 索引
helm repo remove hashicorp             # 移除某個 repository (Remove the `Hashicorp helm repository` from the cluster.)

helm search repo wordpress             # 在本地 repo 索引中搜尋 chart (在已加入的 chart repository 中搜尋)
helm search hub wordpress              # 在 Artifact Hub 上搜尋（需網路）
```

#### Release 安裝與管理
```bash
helm install <release-name> <chart>                       # 安裝 release
helm install <release-name> <chart> --version [x.x.x]     # 安裝 release

helm upgrade --install <release-name> <chart>        # 存在則升級，不存在則安裝
helm rollback <release-name> [revision]              # 回滾 release
helm uninstall <release-name>                        # 移除 release
helm list                                            # 查看所有 release
```

#### 查詢與除錯
```bash
helm get all <release-name>            # 查看 release 詳細內容
helm show values <chart>               # 查看 chart 的可覆蓋變數
helm version                           # 查看 Helm CLI 版本
```

#### 幫助與文件
```bash
helm --help
helm repo --help
helm get --help
helm install --help
helm repo update --help
```

---
# Installation

> [Helm 官方教學](https://helm.sh/docs/intro/quickstart/)  
> 你也可以使用 [get_helm.sh script](https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3) 在任何支援 bash 的系統上安裝。

1. 確保已安裝並啟動 K8s Cluster，可使用 `kubeadm` 部署
2. 驗證 `kubectl`、`kubeadm`、`helm` 版本
	```bash
	kubectl version
	kubeadm version
	```
3. 安裝 Helm 3（適用 Linux/macOS）
	```bash
	curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3
	chmod 700 get_helm.sh
	./get_helm.sh

	helm version
	```

{% note 必要工具安裝 %}
若要使用 Helm，你必須先準備好 Kubernetes Cluster，而 kubectl 和 kubeadm 是協助你「建立」或「連線」這個 Cluster 的工具。

| 工具        | 角色                             | 是否必要 | 說明                                    |
| --------- | ------------------------------ | ---- | ------------------------------------- |
| `kubeadm` | 用來建立 K8s Cluster 的工具           | ❌可選  | 你若使用 EKS、GKE、Minikube 就不需要它           |
| `kubectl` | 與 K8s 溝通的 CLI 工具               | ✅必要  | Helm 執行時會使用 `kubectl` 與 API Server 對話 |
| `helm`    | K8s 應用套件管理器（需要 kube + kubectl） | ✅必要  | 安裝 chart、升級、回滾、解除安裝等功能                |

<br>

✅ 正確順序：
1. （可選）安裝 `kubeadm`
	- → 如果你是自己在本機或 VM 建立 K8s Cluster，就需要 `kubeadm` 來初始化。
2. 建立 Kubernetes Cluster
	- → 使用 `kubeadm init` 或其他方式（如 Minikube、Kind、EKS/GKE）建立一個運作中的叢集。
3. 安裝與設定 `kubectl`
	- → 用來與你的 K8s Cluster 溝通，必須設定 `~/.kube/config` 指向正確的 Cluster。
4. 驗證 Cluster 狀態是否正常（可選）
	```bash
	kubectl get nodes
	kubectl get pods -A
	```
5. 安裝 Helm 並開始部署應用
	- → Helm 利用 `kubectl` 與 cluster 溝通來部署資源。

<br>

🛠️ Cluster 建立工具建議：
| 用途      | 適合工具                            |
| ------- | ------------------------------- |
| 本機學習／開發 | Minikube、Kind、kubeadm           |
| 雲端正式環境  | EKS（AWS）、GKE（Google）、AKS（Azure） |
{% endnote %}

---
# Helm2 vs Helm3
- Helm 1.0: Feb 2016
- Helm 2.0: Nov 2016
	- `+` Tiller
	- `-` 3-Way Strategic Merge Patch
- Helm 3.0: Nov 2019
	- `-` Tiller
	- `+` RBAC (Role Based Access Control)
	- `+` CRD (Custom Resource Definitions) 自定資源
	- `+` 3-Way Strategic Merge Patch

| 功能 / 特性                  | Helm 2                        | Helm 3                         |
| --------------------------- | ----------------------------- | ------------------------------ |
| Server-side component       | ✅ Tiller                     | ❌ 無 Tiller                     |
| 安全性（RBAC）                | ❌ 手動設定                    | ✅ 原生整合                         |
| CRD 支援                     | ⚠️ 需額外處理                  | ✅ 原生支援                         |
| Release 狀態儲存位置          | ConfigMap in Tiller namespace | Secrets/ConfigMap in namespace |
| 3-Way Strategic Merge Patch | ❌                            | ✅                              |


## 3-Way Strategic Merge Patch

當執行 `helm upgrade` 時，Helm 3 使用「三方比對策略」來合併資源變更：
- **Live manifest**：目前實際在 Cluster 中的資源狀態
- **Last applied manifest**：上一次部署（Helm 安裝或升級）所產生的 YAML
- **New manifest**：這次 `helm upgrade` 將套用的新 YAML

Helm 會比較這三份資料的差異，自動計算最安全的更新方式，從而：
- 避免覆蓋使用者透過 `kubectl edit` 所做的即時修改 (保留現有狀態中「使用者手動修改的內容」)
- 提供更穩定、安全的 upgrade 與 rollback 機制

**優點：**
- 安全地進行 `upgrade` 或 `rollback`
- 支援變數參數化與動態部署
- 保留 live 狀態下手動調整的內容
- 提升應用穩定性與維運效率

---
# Helm Charts

Helm Chart 是一個用來描述 Kubernetes 應用的「套件」，包含：
- Kubernetes 資源的 YAML 模板（Templates）
- 預設變數（values）
- Chart metadata（如名稱、版本、依賴）

透過 Helm，你可以將應用「參數化、模組化、版本化」，並用簡單指令完成部署與升級。

## 📁 Chart 目錄結構範例 (hello-world-chart)

A `Chart` is a Helm package. It contains all of the resource definitions necessary to run an application, tool, or service inside of a Kubernetes cluster.

```plaintext
hello-world-chart/
├── Chart.yaml        # Chart metadata：名稱、版本、描述、依賴等
├── values.yaml       # 使用者可自定義變數的預設值
├── templates/        # Kubernetes YAML 模板
│   ├── deployment.yaml
│   ├── service.yaml
│   └── _helpers.tpl  # Template helper function（共用模板變數）
├── charts/           # 子 Chart（可選，用於 dependencies）
├── LICENSE           # 授權條款（可選）
└── README.md         # 說明文件（可選）
```
> ✅ 可用 `helm create hello-world-chart` 快速建立基本結構。

### Chart.yaml 說明
`Chart.yaml` 是每個 Chart 的核心定義檔，描述這個應用的 metadata 與依賴：

```yaml
apiVersion: v2              # Chart API 版本（v1: Helm 2, v2: Helm 3）
name: hello-world           # Chart 名稱
description: A test chart   # 描述文字
type: application           # 類型：application 或 library
version: 0.1.0              # Chart 本身版本（非 App 版本）
appVersion: "1.16.0"        # 實際部署應用的版本
dependencies:               # 依賴的其他 Charts（可選）
  - name: redis
    version: 14.4.0
    repository: https://charts.bitnami.com/bitnami
```

> `apiVersion: v2`
> - Helm 3 使用，支援 dependencies、自動驗證等功能

> type：
> - application（預設）：可直接部署的應用程式（如 NGINX、MySQL）
> - library：提供其他 Chart 引用（不能單獨部署）

> dependencies：
> - 定義需要一併部署的其他 Charts（如資料庫、共用模組）
> - 可手動放進 `charts/` 資料夾或使用 `helm dependency update` 自動下載

### values.yaml 說明

`values.yaml` 是 Helm Chart 的預設變數設定檔，使用者可透過參數調整部署細節。

```yaml
replicaCount: 2

image:
  repository: nginx
  tag: 1.25
  pullPolicy: IfNotPresent

service:
  type: ClusterIP
  port: 80
```

可在 templates/deployment.yaml 中這樣使用：

```yaml
spec:
  replicas: {{ .Values.replicaCount }}

containers:
  - name: app
    image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
    imagePullPolicy: {{ .Values.image.pullPolicy }}
```
> ✅ 使用者可以透過 `helm install -f my-values.yaml` 自訂部署變數。

---
# Customizing Chart Parameters

Helm 提供兩種方式來覆蓋 Chart 中的預設變數設定（values.yaml）：

## 方式一：使用 `--set` 單行指定參數
```bash
helm install --set wordpressBlogName="Helm Tutorials" my-release bitnami/wordpress
```
- 適合修改少量變數。
- 可用 `--set key1=value1,key2=value2` 一行指定多個值。
- 不支援複雜結構（如陣列、巢狀結構）。

## 方式二：使用 `--values` 指定自訂的 YAML 設定檔
custom-values.yaml
```yaml
wordpressBlogName: "Helm YAML Custom Blog"
wordpressUsername: "admin"
wordpressPassword: "admin123"
service:
  type: LoadBalancer
  port: 8080
```
```bash
helm install --values custom-values.yaml my-release bitnami/wordpress
```
- 適合修改多個參數，或巢狀結構。
- 檔案結構需與原本 values.yaml 相容。

## 解壓並修改 Chart 原始檔

你也可以下載並解壓 Helm Chart 原始內容來進一步客製化：
```bash
helm pull bitnami/wordpress
helm pull --untar bitnami/wordpress  # 下載並解壓縮
ls wordpress                         # 瀏覽解壓後的 Chart 目錄
helm install my-release ./wordpress  # 使用本地目錄安裝
```
這方式適合需要修改模板檔（如 deployment.yaml）或加入其他資源。

> 💡 若安裝出現錯誤，請確認 chart repository 已更新：`helm repo update`。

---
# Helm Release LifeCycle Management

使用 Helm 部署後，release 的版本管理與 rollback 是日常操作中很重要的一環。

## 查看 Pod 與部署狀態（K8s 層級）
```bash
kubectl get pods                                  # 查看目前 Pod 狀態
kubectl describe pods [pod_name]                  # 查看指定 Pod 詳細資訊
```

## 升級 Chart（如有更新）
```bash
helm upgrade nginx-release bitnami/nginx          # 升級 nginx-release

helm upgrade dazzling-web bitnami/nginx --version 18.3.6
```
> 若想自訂參數升級，可搭配 `--set` 或 `--values`

## 查看 Release 歷史紀錄
```bash
helm list                                         # 所有 Helm release
helm history nginx-release                        # 查看 nginx-release 的版本歷史
```

## 回滾 Release 至先前版本
```bash
helm rollback dazzling-web

helm rollback nginx-release 1                     # 回到版本 1
```
> Helm 每次 upgrade 都會記錄版本，預設保留最後 10 個版本。
