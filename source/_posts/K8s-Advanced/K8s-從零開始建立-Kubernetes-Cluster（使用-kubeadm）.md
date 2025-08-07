---
title: K8s | 從零開始建立 Kubernetes Cluster（使用 kubeadm）
description: '建立一個從零開始的 Kubernetes Cluster（使用 kubeadm）是學習 Kubernetes 架構與 CKA 考試非常重要的練習。'
date: 2025-08-07 10:41:10
categories: Kubernetes
tags:
- k8s-cluster
---

# 範例架構
| 主機角色 | 主機名        | IP 地址          |
| ------- | ------------ | --------------- |
| master  | `master`     | `172.20.10.10` |
| worker1 | `worker1`    | `172.20.10.11` |

## 系統需求
- Linux OS（建議 Ubuntu 20.04）
- 至少 2GB RAM/VM
- root 權限
- 可以互相 Ping / SSH

## 安裝步驟總覽
1. VirtualBox 開啟兩個 VM
	- Apple M1/M2 (ARM 架構): `ubuntu-24.04.2-live-server-arm64.iso` (Ubuntu Server for ARM64)
2. 設定 IP
	- VirtualBox 網路：使用「一張」介面卡時，請設定「橋接介面卡（Bridged）」、名稱「en0: WiFi」。
3. 設定 SSH
4. 安裝 containerd
5. 安裝 kubeadm, kubelet, kubectl
6. 關閉 Swap（每台都要做）
7. 初始化 Master Node
8. 加入 Worker Node
9. 安裝 Pod Network (如 Calico)

# 🔧 步驟詳細內容（每台都要做）
```
你要在 Ubuntu ARM VM 上手動設定靜態 IP，其中：
- Master 節點 IP 是：172.20.10.10
- Worker1 節點 IP 是：172.20.10.11
```

## 步驟一、二：在 Ubuntu ARM VM 中設定靜態 IP（使用 Netplan）
1. 查詢你的網卡名稱（如 ens33、enp0s3 等）
	```bash
	ip a
	```
2. 編輯 Netplan 設定檔：`/etc/netplan/50-cloud-init.yaml`
	- Netplan 設定檔通常在 `/etc/netplan/` 資料夾內，常見檔案有 `50-cloud-init.yaml`。
	```bash
	ls /etc/netplan/
	sudo vim /etc/netplan/50-cloud-init.yaml
	```
3. 對 Master 設定靜態 IP（172.20.10.10）
	```yaml
	network:
	  version: 2
	  ethernets:
	    ens33:   # <<== 換成你實際的網卡名稱，用 `ip a` 查看
	      dhcp4: no
	      addresses: [172.20.10.10/28]     # <-- 靜態 IP，注意 CIDR
	      nameservers:
	        addresses: [8.8.8.8, 1.1.1.1]
	      routes:
	        - to: default
	          via: 172.20.10.1             # <-- 你主機的閘道
	```
	- ❗️確認三件事：
		- IP 設為：172.20.10.10（不要衝突）
		- 網段（subnet）設為：/28（使用 `ifconfig`，來自 netmask 0xfffffff0）
		- 閘道設為：172.20.10.1
			- MacOS 用 `route -n get default` 查
			- Linux 用 `ip route | grep default`, `route -n`查
	- 🔍 補充：為什麼使用 /28？
		- 從你提供的 Mac 網路資訊 (使用 `ifconfig`)：`inet 172.20.10.2 netmask 0xfffffff0`
			> 這個 netmask（0xfffffff0）對應的是 255.255.255.240，也就是 /28
		- 可用 IP：172.20.10.1 ~ 172.20.10.14（共 14 個）
			> 所以你的靜態 IP 要設定為這個範圍內，且不能跟 Mac 衝突（Mac 是 172.20.10.2）
		- 網段：172.20.10.0/28
4. Worker1 設定靜態 IP（172.20.10.11）
	```yaml
	network:
	  version: 2
	  ethernets:
	    ens33:   # <<== 換成實際的網卡名稱
	      dhcp4: no
	      addresses: [172.20.10.11/28]
	      nameservers:
	        addresses: [8.8.8.8, 1.1.1.1]
	      routes:
		    - to: default
		      via: 172.20.10.1             # <-- 你主機的閘道
	```
5. 套用 Netplan 設定
	```bash
	sudo netplan apply
	ip a          # 看靜態 IP 是否套用
	ping 8.8.8.8  # 測試是否可連上外網
	```

6. 測試 VM 之間是否能互 ping
```bash
# 在 Master 上：
ping 172.20.10.11

# 在 Worker1 上：
ping 172.20.10.10
```

---
## 📌 補充：VirtualBox 網路模式建議
如果你是在 VirtualBox 中操作：
| 模式            | 用途                         |
| ------------- | -------------------------- |
| **Host-Only** | 建議用來讓多台 VM 可互通、且不對外上網      |
| **Bridged**   | 可讓 VM 直接上實體網路，如同實體主機       |
| **NAT**       | 適合上網，但不適合做 kubeadm cluster |

➤ 最推薦你選擇 Host-Only Adapter 模式來組建 Master + Worker 的 K8s Cluster，並手動指定 IP。

| 模式              | 是否能設定靜態 IP  | 是否能從其他裝置連線進來 | 是否能直接上網 | 適用情境                        |
| --------------- | ----------- | ------------ | ------- | --------------------------- |
| **NAT**         | ❌ 不建議       | ❌（需額外轉 Port） | ✅ 可以上網  | 單純上網或測試用                    |
| **橋接（Bridged）** | ✅ 可以（用你家網段） | ✅（就像獨立一台機器）  | ✅ 可以上網  | 多 VM 通訊 / 要靜態 IP / 要被其他裝置存取 |

✅ 若你要設定「靜態 IP」→ 請選「橋接介面卡（Bridged Adapter）」
這樣：
1. 你的 VM 會像是家中網路中的一台「實體機器」
2. 可以直接設定同一網段的 IP，例如你家主機是 172.20.10.2，你 VM 可以是 172.20.10.10
3. 可以跟你 Host（實體機）互 ping、SSH、傳檔案
4. 也可以被其他裝置（同一區網）連線，例如別台電腦、手機

❌ 若選 NAT 模式的話：
- VM 會被「虛擬 NAT 網路」包住（像是躲在牆後面）
- 雖然 VM 可以連外（上網），但別人連不到 VM（除非你手動設 Port Forward）
- VM 也不會拿到跟你實體機相同的網段，無法用 172.20.10.x 做靜態 IP

---
## 步驟三：設定 VM 之間的 SSH 連線
- 目標：
	- 從 VM1（如 master）透過 SSH 連線到 VM2（如 worker），不需要輸入密碼。
- 預備條件：
	- 兩台 VM 都已安裝 Linux、openssh-server
		```bash
		sudo apt update
		sudo apt install openssh-server -y
		sudo systemctl enable ssh
		sudo systemctl status ssh

		# 確認 SSH Port 有監聽
		sudo ss -tuln | grep :22
		#LISTEN  0  128  0.0.0.0:22   0.0.0.0:*  
		```
	- 都能正常連接（互 ping）
	- 你知道對方 IP，例如：
		- VM1 (master)：172.20.10.10
		- VM2 (worker)：172.20.10.11

1️⃣ 在 VM1（控制端）產生 SSH 金鑰
```bash
ssh-keygen -t rsa -b 4096
# 按 Enter 全部預設即可（會生成在 ~/.ssh/id_rsa 和 id_rsa.pub）。
```

2️⃣ 將公鑰複製到 VM2
```bash
ssh-copy-id your_user@172.20.10.11
# 輸入一次密碼，之後即可無密碼登入。
```

3️⃣ 測試 SSH 連線
```bash
ssh your_user@172.20.10.11
# 如果成功直接登入，代表設定完成 ✅
```

---
## 步驟四：安裝 containerd（推薦）
每台 VM 上都需要安裝 container runtime，這裡以 containerd 為例：
```bash
sudo apt update
sudo apt install -y containerd
```

建立預設設定
```bash
sudo mkdir -p /etc/containerd
sudo containerd config default > config.toml
sudo cp config.toml /etc/containerd/

# 修改這一行（使用 systemd cgroup）
sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml

systemctl restart containerd
systemctl enable containerd
```
> 注意：請確保每台 VM 都安裝了 container runtime，並且 `SystemCgroup 也有設定為 true`，否則 cluster 建立後重要元件會不斷重啟而無法使用！

---
## 步驟五：安裝必要組件 kubelet、kubeadm、kubectl
> [官方文件](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/#installing-kubeadm-kubelet-and-kubectl)

在每台 VM（master & worker）上，需要以下三個組件：
1. kubelet：昨天的文章中有提到，它是「小船的船長」。
2. kubeadm：用來部署 cluster 的工具。
3. kubectl：管理員用來與 cluster 進行溝通的 CLI 工具，讓我們能透過下指令來操作 cluster。

```bash
sudo apt-get update
sudo apt-get install -y apt-transport-https ca-certificates curl gpg

# If the directory `/etc/apt/keyrings` does not exist, it should be created before the curl command, read the note below.
# sudo mkdir -p -m 755 /etc/apt/keyrings
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.33/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

# This overwrites any existing configuration in /etc/apt/sources.list.d/kubernetes.list
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.33/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list

sudo apt-get update
sudo apt-get install -y kubelet kubeadm kubectl

# 鎖定版本（防止自動升級導致版本不一致）
sudo apt-mark hold kubelet kubeadm kubectl

# 啟用 kubelet
sudo systemctl enable --now kubelet

kubelet --version
kubeadm version
kubectl version --client
```

---
## 步驟六：關閉 swap 並啟用 ip_forward
在預設上，如果 swap 沒有被關閉，可能會導致 kubelet 無法正常運作。
每台 VM（master & worker）都需要關閉 swap：
```bash
# 關閉 Swap
sudo swapoff -a
sudo sed -i '/ swap / s/^/#/' /etc/fstab    # 將swap的那一行註解掉
```

在 Kubernetes Cluster 中必須啟用 ip_forward（IP 轉送），否則 Pod-to-Pod、Pod-to-Service、跨節點通訊都可能會失敗，特別是在安裝 Flannel、Calico 等 CNI 網路插件時尤為重要。
每台 VM（master & worker）都需要啟用 ip_forward：
```bash
# 永久啟用（建議）
sudo vim /etc/sysctl.conf

# 加入或確保這一行存在：
net.ipv4.ip_forward = 1              # 把註解刪掉

# 然後重新載入設定：
sudo sysctl -p

# 驗證是否成功
cat /proc/sys/net/ipv4/ip_forward    # 輸出為 1 表示已啟用。
```

---
## 步驟七：Master 初始化
在 master 上執行：
```bash
sudo kubeadm init --pod-network-cidr=10.244.0.0/16

sudo kubeadm init \
	--apiserver-advertise-address <master node IP:172.20.10.10> \
	--control-plane-endpoint <master node IP:172.20.10.10> \
	--pod-network-cidr=10.244.0.0/16
```
| Plugin    | 建議的 `--pod-network-cidr` 值 |
| --------- | -------------------------- |
| Flannel   | `10.244.0.0/16`            |
| Calico    | `192.168.0.0/16` 或自訂     |
| Weave Net | 不需要指定（自動處理）         |


⚠️ 初始化完成後重要提醒：
執行下面指令來讓 kubectl 可用：
```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

```bash
kubectl get nodes
# 目前只有 master node 而已
```

---
## 步驟八：Worker 節點加入 Cluster
在 worker 上執行：
```bash
# 從 kubeadm init 結果中會看到類似這段 join 指令，複製它並貼到 worker node 執行：
sudo kubeadm join 172.20.10.10:6443 --token xxxxx \
    --discovery-token-ca-cert-hash sha256:xxxxxxxx
```

---
## 步驟九：Master 安裝 Pod 網路插件（建議用 Calico）
> [Calico 官方文件](https://docs.tigera.io/calico/latest/getting-started/kubernetes/quickstart)
> [K8s - How to implement the Kubernetes network model](https://kubernetes.io/docs/concepts/cluster-administration/networking/#how-to-implement-the-kubernetes-network-model)
> [CNI - 3rd party plugins](https://github.com/containernetworking/cni)

為了讓 cluster 中的 Pod 可以彼此溝通，我們需要安裝 CNI (Container Network Interface) 來部署 Pod network。
推薦安裝 calico，因為它支援的功能較多，包括後續章節會介紹的 NetworkPolicy。
```bash
kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.30.2/manifests/tigera-operator.yaml

# 由於我們在 kubeadm init 時將「--pod-network-cidr」設為 10.244.0.0/16，因此必須先下載 custom-resources 的檔案，修改後再進行安裝：
wget https://raw.githubusercontent.com/projectcalico/calico/v3.27.2/manifests/custom-resources.yaml
vim custom-resources.yaml
kubectl apply -f custom-resources.yaml
```

`custom-resources.yaml`
```yaml
......
spec:
  # Configures Calico networking
  calicoNetwork:
    # Note: The ipPools section cannot be modified post-install.
    ipPools:
    - blockSize: 26
      cidr: 10.244.0.0/16 # 修改這裡!
......
```

等待一段時間後，當 Node 的狀態變成 Ready，就代表 cluster 已經建置完成了。
```bash
watch kubectl get pods -n calico-system

# -w 會持續輸出 node 的狀態
kubectl get nodes -w
#NAME       STATUS   ROLES           AGE   VERSION
#master     Ready    control-plane   10m   v1.33.3
#worker1    Ready    <none>          2m    v1.33.3

watch kubectl get tigerastatus
```

## 未來加入新的 Worker Node
如果在未來需要加入新的 Worker Node 到 Master 中，在 Master Node 上執行以下指令：
```bash
kubeadm token create --print-join-command
```
將上述指令的輸出複製並在新的 Worker Node 上執行，就可以加入 cluster 了。

---
# Quick Start
以下是使用 kubeadm 建立的 Kubernetes Cluster 中部署 Nginx 的最簡教學範例，包含完整步驟與驗證指令，適合初學者練習。

## 步驟一：建立 Nginx Deployment
這會建立一個名稱為 nginx-deployment 的 Deployment，使用官方穩定版的 Nginx 映像檔。
```bash
kubectl create deployment nginx-deployment --image=nginx:stable
```

查看部署狀態
```bash
kubectl get deployments
kubectl get pods -o wide
```

## 步驟二：建立 `NodePort` Service 供外部訪問
這會自動建立一個 Service，連接到 nginx-deployment 的 Pod 並開放一個 NodePort（預設在 30000~32767 範圍）。
```bash
kubectl expose deployment nginx-deployment --type=NodePort --port=80 --name=nginx-service
```

## 步驟三：驗證部署結果
查詢 NodePort 端口
```bash
kubectl get svc nginx-service
```

你應該會看到像這樣的結果：
```
NAME             TYPE       CLUSTER-IP     EXTERNAL-IP   PORT(S)        AGE
nginx-service    NodePort   10.108.225.160  <none>        80:31653/TCP   1m
```

這表示你可以用 `終端機輸入` 或 `瀏覽器開啟` 方式，存取 Nginx：
```bash
# 終端機輸入
curl http://<任何一台 Worker Node 的 IP>:31653

# 瀏覽器開啟
http://<worker-node-ip>:31653 查看 Nginx 的歡迎頁面。
e.g. http://172.20.10.11:31653/

如果出現 Nginx 歡迎頁，就表示部署成功！
```

