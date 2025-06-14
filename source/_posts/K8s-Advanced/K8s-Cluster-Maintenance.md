---
title: K8s | Cluster Maintenance
description: ' '
date: 2025-06-13 20:54:36
categories: Kubernetes
tags:
- OS Upgrades
- Backup
- Restore
---

# OS Upgrades

## 重要指令

### kubectl drain \<NODE_NAME>
```bash
kubectl drain node01 --ignore-daemonsets
```
將 K8s Cluster 中的 node01 節點進行「排空」操作，也就是把 node01 上的 Pod (所有非 DaemonSet 的 Pod) 移開或刪除，以便進行維護或更新。

{% note default "補充說明：kubectl drain" %}
<mark>預設會做什麼？</mark>
1. 將節點標記為 unschedulable：
    - 相當於執行 `kubectl cordon node01`
    - 表示新的 Pod 不會再被排程到這個 node 上
2. 驅逐(刪除) old node 上的非 DaemonSet Pod：
    - 將這些 Pod 標記為 Evicted，並由其 Controller（如 Deployment、StatefulSet）自動在其他節點重新部署
3. 不處理以下 Pod 類型 (除非加入額外參數)：
    - DaemonSet Pod (除非加入 --delete-emptydir-data)
    - Mirror Pod (靜態 Pod)
    - Unmanaged Pod（沒有 controller，例如未由 ReplicaSet 管理的 Pod）

<mark>相關補充：</mark>
1. `--grace-period=SECONDS`（Pod 優雅終止時間）
    - 影響單一 Pod 被刪除前等待多久讓其優雅終止（執行清理邏輯）。
    - 預設值：來自 Pod 的 terminationGracePeriodSeconds（通常為 30 秒）。
    - ✅ 可透過 `kubectl drain node01 --grace-period=10` 指定。
2. `--pod-eviction-timeout=TIME`（節點失效時的驅逐等待時間）
    - 是 kube-controller-manager 的參數，控制當節點 NotReady 或 Unreachable 時，K8s 等多久才會驅逐該 node 上的 Pod。
    - 預設值：5 分鐘。
    - ✅ 僅在節點異常（自動驅逐）時生效，與 kubectl drain 無關。

<mark>常見用途：</mark>
1. 節點維修（硬體更換、OS 升級）
2. 安全性修補（Patching）
3. 節點關機或重啟
4. 滾動更新 OS 或 kubelet
{% endnote %}

### kubectl cordon \<NODE_NAME>
```bash
kubectl cordon node02
```
將 node02 標記為 unschedulable (只封鎖排程)，不移除任何 Pod。

{% note default "補充說明：kubectl cordon" %}
<mark>使用情境：</mark>
- 當你只想停止新的 Pod 被安排到此 node，但不想驅逐目前 node 上的 Pod 時。
{% endnote %}

### kubectl uncordon \<NODE_NAME>
```bash
kubectl uncordon node01
```
解除 node 封鎖 (恢復可調度狀態)，允許排程 (新的 Pod 被 schedule 到 node 上)。

{% note default "補充說明：kubectl uncordon" %}
<mark>使用情境：</mark>
- 你已經完成 node01 的維護作業 (例如系統更新、重啟、硬體修復)。你想要讓 node01 恢復正常調度時。
{% endnote %}

### Summary
{% note info %}
**背景知識：**
1. *當你使用 `kubectl drain node01` 排空 node 時，它會同時把 node 標記為不可調度（unschedulable），以避免有新 Pod 被分派至該機器。*
2. *使用 uncordon 則是「恢復調度」功能，讓 node 可以再次承擔新的 Pod 排程任務。*

**總結：**
- `kubectl drain` = 封鎖排程 + 驅逐非 DaemonSet Pod
- `kubectl cordon` = 僅封鎖排程，不移動 Pod
- `kubectl uncordon` = 恢復排程功能

**建議搭配下面指令觀察節點狀態：**
```bash
kubectl get nodes -o wide
kubectl describe node node01
```
{% endnote %}

在 Kubernetes 中進行 OS 升級（例如 Ubuntu、CentOS、RHEL、Amazon Linux 等）時，為了最小化服務中斷，建議採取 **滾動升級（Rolling Upgrade）** 策略，確保集群持續可用。

1. 升級前準備
    - 檢查節點狀態
      `kubectl get nodes -o wide`
    - 標記節點為不可調度（防止新 Pod 被排到這台機器）
      `kubectl cordon <NODE_NAME>`
    - 排空節點（移除現有 Pod）
      `kubectl drain <NODE_NAME> --ignore-daemonsets --delete-emptydir-data`
      > --ignore-daemonsets：保留 DaemonSet pod
      > --delete-emptydir-data：允許刪除使用 emptyDir 的 Pod
2. 升級作業系統
    - Ubuntu 範例
      ```bash
      sudo apt update && sudo apt upgrade -y
      sudo reboot
      ```
3. 重啟後驗證與重新加入節點
    - 驗證節點狀態：節點應該會從 NotReady 回到 Ready 狀態。
      `kubectl get nodes`
    - 允許排程回該節點
      `kubectl uncordon <NODE_NAME>`
4. 針對其他節點重複此流程
    - 建議依序進行每個節點，確保系統與服務不中斷。

---
# K8s Releases

## 重要指令

```
kubectl get nodes
```

## 版本解析
> [Kubernetes/releases](https://github.com/kubernetes/kubernetes/releases)

v1.11.3
- v1: Major (重大架構變動，極少變動)
- 11: Minor (Features, Functionalities)
- 3:  Patch (Bug Fixes)

## K8s component 升級策略 -- 版本選擇

<mark>kubeadm 的版本必須「等於或高於」你要升級的 Kubernetes cluster 版本</mark>
<mark>Kubernetes 官方僅支援 `最近三個「Minor」版本`（N、N-1、N-2）</mark>
| 最新版本（N） | 同時支援的版本    |
| ------- | ------------------- |
| v1.30   | v1.30, v1.29, v1.28 |
| v1.29   | v1.29, v1.28, v1.27 |

也就是說：
- <mark>只有最近三個 minor 版本會獲得官方的 patch 更新與安全修復。</mark>
- 每個版本的支援週期約為 1 年（每個版本約每 3~4 個月發佈一次）。

{% note info %}
ETCD CLUSTER
- 通常由 kubeadm upgrade 管理，支援升級前備份
- 版本不能跳太多，應查閱官方 [etcd 升級指南](https://etcd.io/docs/v3.6/upgrades/)

CoreDNS
- 升級過程中 kubeadm 會提示是否升級
- 使用者需手動確認是否升級

Kubernetes 元件版本相容性 (以 kube-apiserver 版本為準，X = v1.10)
| 元件                          | 建議版本範圍                     |
| ----------------------------- | ----------------------------- |
| Controller-manager、kube-scheduler | X 或 X-1（例：v1.9 或 v1.10）    |
| kubelet、kube-proxy           | X、X-1、X-2（例：v1.8 或 v1.9 或 v1.10） |
| kubectl                       | X-1、X、X+1（例：v1.9 或 v1.10 或 v1.11） |
{% endnote %}

---
# Cluster Upgrade Process

在 Kubernetes 的升級流程中，應該先從 Control Plane（master 節點）開始升級，再升級 worker 節點。
- [Docs: Upgrading control plane nodes](https://kubernetes.io/docs/tasks/administer-cluster/kubeadm/kubeadm-upgrade/)
- [Docs: Upgrading worker nodes](https://v1-32.docs.kubernetes.io/docs/tasks/administer-cluster/kubeadm/upgrading-linux-nodes/)

<mark>在 Kubernetes 中，升級（Upgrade）是單向的流程，一旦升級完成就 ⚠️ 無法直接降版（Downgrade）。</mark>

## 重要指令

### kubectl get nodes
```bash
kubectl get nodes
```
Get current version of the cluster.

### kubeadm upgrade plan
```bash
kubeadm upgrade plan
```
查看目前升級計畫
可知 Cluster version、kubeadm version、Latest stable version ...

### kubeadm upgrade apply
```bash
kubeadm upgrade apply
```
執行升級，升級 Control Plane 節點（只做 master）

## ✅ 升級順序

1. [升級 Control Plane（Master）節點](https://v1-32.docs.kubernetes.io/docs/tasks/administer-cluster/kubeadm/kubeadm-upgrade/)
    - 包括 kube-apiserver、controller-manager、scheduler、etcd
    Step 0：前置檢查
    ```bash
    kubectl get nodes
    kubectl version
    kubeadm version
    vim /etc/apt/sources.list.d/kubernetes.list
    ```
    Step 1：升級 kubeadm 工具（本機）
    ```bash
    # 更新 apt 索引
    sudo apt-get update
    # 列出你目前系統從 apt 套件來源可以抓到的所有 kubeadm 版本
    sudo apt-cache madison kubeadm

    # 安裝指定版本 kubeadm
    sudo apt-get install -y kubeadm='1.32.0-1.1'
    ```
    Step 2：模擬升級計畫 (預覽) + 套用升級 
    ```bash
    kubeadm upgrade plan
    kubeadm upgrade apply v1.32.0
    ```
    Step 3：升級 kubelet 與 kubectl
    ```bash
    sudo apt-get update
    sudo apt-get install -y kubelet='1.32.0-1.1' kubectl='1.32.0-1.1'
    sudo systemctl daemon-reload
    sudo systemctl restart kubelet
    ```
    Step 4：Verify the status of the cluster
    ```bash
    kubectl get nodes -o wide
    ```
2. [升級 Worker 節點](https://v1-32.docs.kubernetes.io/docs/tasks/administer-cluster/kubeadm/upgrading-linux-nodes/)
    - 包括 kubelet、kube-proxy、OS
    Step 0：排空節點 (Drain)
    ```bash
    kubectl drain <worker-node> --ignore-daemonsets
    ```
    Step 1：進入該 Worker 節點，升級 kubeadm 工具
    ```bash
    ssh node01

    # 確保來源正確（只需設定一次）
    vim /etc/apt/sources.list.d/kubernetes.list

    # 更新套件資訊並安裝 kubeadm v1.32.0
    sudo apt-get update && sudo apt-get install -y kubeadm='1.32.0-1.1'

    # 查實際版本
    apt-cache madison kubeadm
    ```
    Step 2：升級 kubeadm
    對於 Worker 節點，只需要這個命令即可（不需 apply，那是 Control Plane 用的）
    ```bash
    sudo kubeadm upgrade node
    ```
    Step 3：升級 kubelet 與 kubectl 並重啟服務
    ```bash
    sudo apt-get update && sudo apt-get install -y kubelet='1.32.0-1.1' kubectl='1.32.0-1.1'
    sudo systemctl daemon-reload
    sudo systemctl restart kubelet
    exit
    ```
    Step 4：Verify the status of the cluster (on Control Plane node)
    ```bash
    kubectl get nodes -o wide
    ```
    Step 5：取消排空 (Uncordon)
    ```bash
    kubectl uncordon <worker-node>
    ```

{% note info %}
**為什麼要先升級 Control Plane？**
- Control Plane 決定 Cluster API 行為與版本邏輯，所有 worker 節點都必須與 apiserver 相容。
- worker 節點允許落後最多 2 個版本（X-2），但 Control Plane 通常不能落後 worker。
- 先升級 worker 可能導致不相容的 kubelet 版本 → apiserver 拒絕通信。

**實務注意事項**
- 多個 master 節點時，`一台一台升級`，可保持 HA（高可用）不中斷。
- 可透過 kubectl get nodes 檢查版本，搭配 kubeadm upgrade plan 來規劃升級路線。
- 升級前 `備份 etcd` 是很好的保險措施。
{% endnote %}

## Upgrade Strategy (3 種方式)

1. 一次升級所有節點（會停機）
    - 適合情境：測試環境、非高可用架構
    - 優點：快速簡單
    - 缺點：服務停機
2. 輪流升級現有節點 (不中斷服務)
    > 一個 Upgrade 完，再換下一個 Upgrade (沒有新增 node，直接用現有的 node 進行)
    - 適合情境：小規模、可接受稍微波動
    - 優點：不需擴容
    - 缺點：升級期間有壓力
3. 新節點接手、移除舊節點 (先加新節點升級 → 移動應用 → 移除舊節點)
    > 先新增一個 Upgrade 後的 node，把 application 放到該 Upgrade 後的 node 上運行，再把舊 node 砍掉 (有新增 node)
    - 適合情境：大型企業、高可用要求
    - 優點：無中斷、可測試新節點
    - 缺點：成本較高、需備援資源

---
# Backup and Restore

- Backup 範圍
  - Resource Configuration：所有 K8s 資源設定檔（Deployment、Service、Secrets）
  - ETCD Cluster：儲存整個 Cluster 狀態（control plane source of truth）
  - Persistent Volumes：PV/PVC（取決於是否使用支援 snapshot 的 StorageClass）

## 重要指令

### Backup - Resource Configuration

#### ✅ Imperative 建立資源
```
kubectl create namespace <name>

kubectl create secret generic <name> --from-literal=password=mypassword

kubectl create configmap <name> --from-literal=key=value
```

#### ✅ Declarative 建立資源
```yaml
# pod-definition.yml

apiVersion: v1
kind: Pod
metadata:
  name: myapp-pod
  labels:
    app: myapp
    type: front-end
spec:
  containers:
    - name: nginx-container
      image: nginx
```
```bash
kubectl apply –f pod-definition.yml
```

#### ✅ 將所有資源匯出成 YAML 🌟
```bash
kubectl get all --all-namespaces -o yaml > all-resources.yaml
kubectl get configmap,secret --all-namespaces -o yaml > config-secret.yaml
```
```bash
# 補充：也可以逐一備份各資源種類
kubectl get deployments.apps --all-namespaces -o yaml > deployments.yaml
kubectl get svc --all-namespaces -o yaml > services.yaml
```

### Backup - ETCD (ETCD Cluster) Snapshot

#### ✅ 方式一：使用 etcdctl（Snapshot-based Backup，推薦）🌟
> To take a snapshot from a running etcd server
```bash
ETCDCTL_API=3 etcdctl \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  snapshot save /backup/etcd-snapshot.db
```

範例：
```bash
kubectl describe pod etcd-controlplane -n kube-system

# Command:
#       etcd
#       --advertise-client-urls=https://192.168.42.128:2379
# V     --cert-file=/etc/kubernetes/pki/etcd/server.crt
#       --client-cert-auth=true
#       --data-dir=/var/lib/etcd
#       --experimental-initial-corrupt-check=true
#       --experimental-watch-progress-notify-interval=5s
#       --initial-advertise-peer-urls=https://192.168.42.128:2380
#       --initial-cluster=controlplane=https://192.168.42.128:2380
# V     --key-file=/etc/kubernetes/pki/etcd/server.key
#       --listen-client-urls=https://127.0.0.1:2379,https://192.168.42.128:2379
#       --listen-metrics-urls=http://127.0.0.1:2381
#       --listen-peer-urls=https://192.168.42.128:2380
#       --name=controlplane
#       --peer-cert-file=/etc/kubernetes/pki/etcd/peer.crt
#       --peer-client-cert-auth=true
#       --peer-key-file=/etc/kubernetes/pki/etcd/peer.key
#       --peer-trusted-ca-file=/etc/kubernetes/pki/etcd/ca.crt
#       --snapshot-count=10000
# V     --trusted-ca-file=/etc/kubernetes/pki/etcd/ca.crt

ETCDCTL_API=3 etcdctl \
  snapshot save /opt/snapshot-pre-boot.db \
  --cacert /etc/kubernetes/pki/etcd/ca.crt \
  --cert /etc/kubernetes/pki/etcd/server.crt \
  --key /etc/kubernetes/pki/etcd/server.key
```
- `--endpoints`: Optional Flag, points to the address where ETCD is running (127.0.0.1:2379)
- `--cacert` Mandatory Flag (Absolute Path to the CA certificate file)
- `--cert` pMandatory Flag (Absolute Path to the Server certificate file)
- `--key` Mandatory Flag (Absolute Path to the Key file)

確認快照狀態：
```bash
ETCDCTL_API=3 etcdctl snapshot status /backup/etcd-snapshot.db \
  --write-out=table
```

#### ✅ 方式二：使用 etcdutl（File-based Backup，需停機）
> For offline file-level backup of the data directory
```bash
etcdutl backup \
  --data-dir /var/lib/etcd \
  --backup-dir /backup/etcd-backup
```

### Restore - ETCD (ETCD Cluster) Snapshot
⚠️ 還原動作會覆蓋原有資料，請審慎執行！

#### ✅ 還原 snapshot（建立新 etcd data-dir）
> To restore a snapshot to a new data directory
```bash
# 停止 kube-apiserver，避免競爭 etcd socket
systemctl stop kube-apiserver


# 使用 etcdctl 還原 snapshot 到新目錄（不覆蓋原目錄）
ETCDCTL_API=3 etcdctl snapshot restore /backup/etcd-snapshot.db \
  --data-dir /var/lib/etcd-from-backup \
  --initial-cluster master-1=https://192.168.5.11:2380,master-2=https://192.168.5.12:2380 \ → 多節點（HA etcd cluster）
  --initial-cluster-token etcd-cluster-1 \
  --initial-advertise-peer-urls https://192.168.5.11:2380


# 修改 etcd.yaml (Static Pod) 使用新的 data-dir（/var/lib/etcd-from-backup）
vim /etc/kubernetes/manifests/etcd.yaml
# 找到 --data-dir 改為 /var/lib/etcd-from-backup
  # ...
  # volumes:
  # - hostPath:
  #     path: /var/lib/etcd-from-backup # New restored backup directory
  #     type: DirectoryOrCreate
  #   name: etcd-data


# 重載並觀察 etcd / kube-apiserver 重啟狀況
systemctl daemon-reload
systemctl restart etcd
systemctl start kube-apiserver


# 監看容器啟動
watch crictl ps
kubectl get deployments,services --all-namespaces
```

{% note info %}
Check the image used by the ETCD pod
```bash
# Method 1
kubectl describe pod etcd-controlplane -n kube-system

# Method 2
kubeadm upgrade plan
```
{% endnote %}

---
# K8s Architecture

- Master
  - 用途：Manage, Plan, Schedule, Monitor Nodes
  - 元件：ETCD CLUSTER, kube-apiserver, kube-scheduler, Kube Controller Manager
- Worker Nodes
  - 用途：Host Application as Containers
  - 元件：kubelet, Kube-proxy, Container Runtime Engine (如 containerd)
