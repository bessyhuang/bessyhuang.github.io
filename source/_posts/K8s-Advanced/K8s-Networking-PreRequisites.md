---
title: K8s | Networking - PreRequisites
description: '深入解析 Kubernetes 網路基礎與 CNI（Container Network Interface）運作方式，透過 veth 與 netns 實作模擬容器網路連線，協助理解 CNI Plugin 如何建立 Pod-to-Pod 通訊，為 CKA 考試打下基礎。'
date: 2025-07-24 09:28:00
categories: Kubernetes
tags:
- Networking
- Pre-Requisites
- CNI
---

# Pre-Requisites – Network, Switching, Routing, Tools

```
[PC1]---(Switch)---[PC2]        # Switching
[LAN]---(Router)---[Internet]   # Routing & NAT
```

## Switching（交換）
- 功能：在**同一個區域網路（LAN）內**傳送資料。
- 設備：Switch（交換器）
- 工作層級：OSI 第 2 層（Data Link Layer）
- 根據什麼來轉發資料？ ➤ 根據 **MAC 位址**
- 常見用途：
	- 建立**多個主機之間**的區域網路連線。
	- 每個埠都有獨立帶寬，減少碰撞（collision domain 獨立）。
- MAC Address Table（MAC 位址表）：
	- Switch 會學習每個 MAC 位址來自哪個 port，根據此表轉發封包。

```
範例：PC1 要傳資料給 PC2，交換器查詢 MAC 表決定送出方向。

[PC1:192.168.1.10] <--> [PC2:192.168.1.11]
```

```bash
# 顯示系統中所有網路裝置，例如 eth0, lo（loopback）。顯示裝置狀態（UP/DOWN）、MAC 位址等。
ip link

# 顯示每個網路介面（interface）的 IP 設定。
ip addr
ifconfig

# 給 eth0 指定一個靜態 IP：192.168.1.10，子網路為 /24。 /24 代表子網路遮罩是 255.255.255.0。
ip addr add 192.168.1.10/24 dev eth0
# 同樣方法設定第二台機器的 IP
ip addr add 192.168.1.11/24 dev eth0

# 驗證是否可以從 192.168.1.10 ping 通對方，證明交換器或虛擬橋接正常運作。這是在同一網段下，不經過路由的傳送。
ping 192.168.1.11
```

## Routing（路由）
- 功能：在**不同網路之間**轉送資料。
- 設備：Router（路由器）
- 工作層級：OSI 第 3 層（Network Layer）
- 根據什麼來轉發資料？ ➤ 根據 **IP 位址**
- 常見用途：
	- 公司網路**連接外部網際網路**。
	- 家用路由器將家中多設備連上網際網路。
	- 在**多子網之間**傳遞資料。
	- 將 LAN、WAN 之間的資料傳送到正確的網段。
- 路由表（Routing Table）：
	- 儲存目的網段與下一跳（next hop）關係。
	- 可設定靜態路由或啟用動態路由協定（如 OSPF、BGP 等）。

```
範例：家中路由器連接兩個內網子網段（192.168.1.0/24 與 192.168.2.0/24），路由器負責轉發這兩個子網之間的流量。

[PC1:192.168.1.10] <--> [Router: 192.168.1.1 | 192.168.2.1] <--> [PC2:192.168.2.20]
```

```bash
# 顯示目前系統的路由表。列出 default gateway、直連路由（connected routes）等資訊。
ip route

# 加入靜態路由規則
# 表示：「若要前往 192.168.2.0/24，下一跳是 192.168.1.1」。注意：設定要通往整個 192.168.2.0/24 子網的路由規則
ip route add 192.168.2.0/24 via 192.168.1.1

# 刪除路由規則
ip route del 192.168.2.0/24
```

```bash
# 模擬兩個子網的路由器功能（實驗環境）
# 如果你在本機使用 network namespace 模擬環境，可建立一個充當路由器的節點：

# 開啟 IP forwarding（Linux 預設關閉）
echo 1 > /proc/sys/net/ipv4/ip_forward
```

## Default Gateway（預設閘道）
- 功能：主機在無法找到目的 IP 所在網段時，將封包送到「預設閘道（Default Gateway）」處理，讓路由器決定後續的傳送路徑。
- 常見設定：
	- 每台主機通常只設一個 Default Gateway。
	- 預設閘道的 IP 通常是路由器的 IP（例如：192.168.1.1）。
	- 若無設置預設閘道，主機將無法存取其他網段（包括上網）。
- 當資料要送往 Google 或其他外部網站時，封包會先被送往 Default Gateway，由路由器轉發到下一跳或網際網路。

```bash
# 範例：你的電腦（192.168.1.10）要連到 Google（8.8.8.8），因為 8.8.8.8 不在 192.168.1.0/24 網段內，
# 所以電腦會把封包送到 Default Gateway（192.168.1.1），讓路由器幫忙找到 Google 的 IP 路徑。

# 查看目前設定的預設路由（即 default gateway）。
ip route show default
```

```bash
# 若已有預設路由，需先刪除原本的。注意：Linux 中只允許一條主要的 default route（可以有多條但要設定優先順序 metric）。
ip route del default
# 新增一條預設路由規則，透過 192.168.2.1 作為下一跳。
ip route add default via 192.168.2.1
```

| 類型            | 功能說明             | 指令範例                                |
| ------------- | ---------------- | ----------------------------------- |
| Default Route | 處理所有不在路由表中的目的 IP | `ip route add default via <IP>`     |
| Static Route  | 處理特定目的網段的封包      | `ip route add 10.0.0.0/24 via <IP>` |


## NAT（Network Address Translation）
- 功能：將內部私有 IP 和外部公有 IP 互相轉換。
- 用途：
	- 讓多台內部設備共用一個公有 IP。
	- 強化安全性，避免內部 IP 直接暴露。
- 常見類型：
	- SNAT（Source NAT）：轉換來源 IP（內 → 外）
	- DNAT（Destination NAT）：轉換目的 IP（外 → 內）
	- PAT（Port Address Translation）：多台內部主機共用一個公網 IP + 不同的 port
- 在 K8s 中，NAT 常出現在：
	- ClusterIP Service: SNAT
	- NodePort: DNAT
	- ExternalTrafficPolicy=Local: 會影響是否做 NAT
- 常見私有 IP 範圍（不能在 Internet 上直接使用）：
	- `10.0.0.0/8`
	- `172.16.0.0/12`
	- `192.168.0.0/16`

> 範例：你在家中開瀏覽器（192.168.0.5）訪問網站，路由器會將你的私有 IP 替換成公有 IP。

## 總結比較表
| 項目     | Switching           | Routing           | Default Gateway         | NAT                           |
| ------ | ------------------- | ----------------- | ----------------------- | ----------------------------- |
| 功能     | 同網段內資料傳送            | 不同網段間資料傳送         | 主機不知去向的流量所使用的預設出口（轉送非本地 IP 封包）    | 私有 IP ↔ 公有 IP 轉換（進出網際網路）      |
| 依據     | MAC 位址              | IP 位址             | 主機 IP 設定中指定的閘道          | 封包來源或目的 IP（SNAT/DNAT）         |
| OSI 層級 | Layer 2 (Data Link) | Layer 3 (Network) | Layer 3                 | Layer 3（IP 層，常與 conntrack 配合追蹤連線狀態）        |
| 設備     | Switch（交換器）         | Router（路由器）       | Router / Layer 3 Switch | Router / Firewall / NAT Gateway |

| 主題              | 是否為 CKA 重點 | 補充說明                                                           |
| --------------- | ---------- | -------------------------------------------------------------- |
| Switching       | ❌ 不是       | Pod 在同一 Node 間溝通，透過 Linux bridge（如 cni0），為 Layer 2 概念，CKA 不深入考    |
| Routing         | ✅ 中等       | Pod 跨 Node 的流量、Service 的 ClusterIP 與 kube-proxy 都涉及 L3 routing |
| Default Gateway | ✅ 中等       | Node 如何將 Pod 的 egress 流量送到外部網路，需理解 default route 設定            |
| NAT             | ✅ 高        | SNAT 用於 Pod 出網（source NAT）、DNAT 則將虛擬 IP 轉向 Pod 真實 IP，kube-proxy 實作時常見 |


| 概念              | 層級      | 根據什麼轉發    | 傳送範圍        | 常見工具                                    |
| --------------- | ------- | --------- | ----------- | --------------------------------------- |
| Switching（交換）   | OSI L2  | MAC 位址    | 同一個 LAN     | `ip link`, `ip addr`, `bridge link`<br>檢查/設定網卡與 L2 網橋狀態    |
| Routing（路由）     | OSI L3  | IP 位址     | 跨子網/跨網段     | `ip route`, `ip forward`, `traceroute`<br>查看路由表、啟用轉發、追蹤封包路徑 |
| Default Gateway | 路由設定一部分 | 預設轉送非本地流量 | 轉送至外部或上層路由器 | `ip route add default`, `ip route show`<br>設定與查詢 default gateway |

> **Pod 間跨 Node 通訊的基礎**：K8s CNI 插件會自動設好路由規則。

## Kubernetes 實務關聯說明
Kubernetes 的網路抽象背後建立在傳統網路知識上。掌握 Switching、Routing、Default Gateway 與 NAT 等概念，能幫助你更有效排查錯誤、設定 CNI、理解 kube-proxy 運作細節。

為何這些基礎知識重要？
- Routing & NAT 是 Kubernetes 網路模型（尤其是 CNI）的核心概念。
	- Kubernetes 中，Pod 同 Node 間的通訊，透過 veth pair 與 Linux bridge（如 cni0）建立橋接，實際上屬於 Layer 2 Switching。
	- Kubernetes 設計中，每個 Pod 都會被分配一個 routable IP，跨 Node 的 Pod 通訊藉由每個 Node 上的路由表與 CNI 插件實現。
- NAT 的理解 有助於排查 Service 無法連線的問題（SNAT/DNAT 錯誤）。
	- 例如使用 NodePort 或 External IP 進行連線時，會透過 DNAT 將外部請求轉向對應的 Service Cluster IP，再由 kube-proxy 分流至實際 Pod。
- Default Gateway 涉及 Pod egress 到外部網路，或與外部 Service 溝通的能力。
- Switching 雖不考，但理解基本 L2 溝通，有助於 NetworkPolicy 與 Bridge 網路除錯。

## DNS
什麼是 DNS（Domain Name System）？
> 將「人類記得住的網域名稱（如 google.com）」轉換為「機器可以識別的 IP 位址（如 142.250.66.132）」。

### 名稱解析流程（Name Resolution）
1. 本地 `/etc/hosts` 檔案
2. DNS Server（依 `/etc/resolv.conf`）
	```yaml
	# /etc/resolv.conf 範例
	nameserver 8.8.8.8
	nameserver 1.1.1.1
	```
3. 依 `/etc/nsswitch.conf` 設定的順序解析
	```yaml
	# /etc/nsswitch.conf 中的 hosts 配置範例如下：
	hosts: files dns myhostname

	# files：代表先查 /etc/hosts
	# dns：再查 DNS Server
	# myhostname：允許系統自己認得 hostname（非必要，但某些 Linux 發行版會加）
	```

```bash
cat /etc/hosts
ping db
ssh db
curl http://www.google.com/

cat /etc/resolv.conf

# 查詢名稱解析順序
cat /etc/nsswitch.conf | grep hosts
# 通常為：hosts: files dns

# 顯示當前主機的名稱
hostname
```

```bash
# 如果 ping db 失敗，可以這樣查原因：
cat /etc/hosts
cat /etc/resolv.conf
cat /etc/nsswitch.conf
```

### DNS 查詢流程範例
Domain Name 結構：
```
apps.google.com
├── apps       # Subdomain (e.g. maps, www, drive, apps)
├── google     # Second Level Domain
├── com        # Top-Level Domain (e.g. .com, .net, .edu, .org, .io)
└── .          # Root (隱含)
```

apps.google.com 查詢流程：
```
[Client] ─┬─> /etc/hosts ⭐️
          ├─> /etc/resolv.conf → Org DNS
          │                   └─> Cache hit? → 回應
          │                   └─> Cache miss →
          │                                 Root DNS
          │                                   ↓
          │                                 .com DNS
          │                                   ↓
          │                             google.com NS
          │                                   ↓
          │                          google DNS Server
          │                                   ↓
          │                           216.58.221.78
          └─<─ 回傳並快取 (Cache 結果到 Org DNS 或本機)
```

### DNS Record Types
| 類型    | 說明                           | 範例                                                       |
| ----- | ---------------------------- | -------------------------------------------------------- |
| A     | 網域名稱 → IPv4 位址               | `A www.google.com → 142.250.66.132`                      |
| AAAA  | 網域名稱 → IPv6 位址               | `AAAA www.google.com → 2607:f8b0::`                      |
| CNAME | Canonical Name（別名）           | `CNAME mail → ghs.google.com`                            |
| MX    | Mail eXchanger（郵件交換伺服器）      | `MX google.com → smtp.google.com`                        |
| TXT   | 附加文字，常用於驗證（如 SPF、DKIM）       | `TXT google.com → v=spf1 include:_spf.google.com`        |
| NS    | 指定此網域使用的 DNS 名稱伺服器           | `NS google.com → ns1.google.com`                         |
| SOA   | Start of Authority，定義網域的主要資訊 | 包含主 DNS、序號、TTL、負責人 Email 等                               |
| PTR   | IP → 網域名稱（反查用）               | `PTR 132.66.250.142.in-addr.arpa → google.com`           |
| SRV   | 提供服務的位址和連接埠                  | `SRV _sip._tcp.example.com → sipserver.example.com:5060` |

### DNS 查詢工具補充
| 工具           | 說明                      | 常用指令範例                                            |
| ------------ | ----------------------- | ------------------------------------------------- |
| `nslookup`   | 傳統工具，只查詢 `DNS Server` 記錄  | `nslookup www.google.com`                         |
| `dig`        | 現代工具，支援多種記錄與查詢細節        | `dig www.google.com A` 、 `dig +short google.com`  |
| `host`       | 簡潔快速查詢                  | `host www.google.com` 、 `host 8.8.8.8`（反查 IP）     |
| `ping`       | 同時測試網路連線與名稱解析           | `ping google.com`                                 |
| `traceroute` | 顯示封包路徑                  | `traceroute google.com` 或 `tracepath`（Linux 上較常見） |
| `curl`       | 測試 HTTP 請求與 DNS 是否能解析成功 | `curl -I https://google.com`                      |

---
# Pre-Requisites – Network Namespaces

## 什麼是 Network Namespace？
Network Namespace（NetNS）是 Linux 核心支援的虛擬網路隔離機制，可讓不同容器、Pod 擁有各自獨立的網路堆疊。
- 每個 Namespace 擁有自己的：
	- Routing Table
	- Network Interfaces
	- IP Address Table
	- ARP Cache
	- `iptables` (部分隔離)
> ✅ **Kubernetes 的每個 Pod、Docker Container 就是透過 NetNS 實現網路隔離！**

## 核心概念
1. Network Namespace 是容器網路的基礎
	- 每個容器（或 Pod）都運作在自己的 Network Namespace。
	- NetNS 提供隔離的網路堆疊（IP、路由表、ARP、iptables…），是實現多容器網路分離的基礎技術。
2. veth pair 建立 namespace 之間的連線
	- veth pair 像是「虛擬網路線」：一端在 Pod/namespace，另一端連接 bridge（虛擬交換器）。
	- 用來模擬 Pod 與 Pod、Pod 與 host 間的網路通訊。
3. Linux Bridge 作為虛擬交換器
	- 可連接多個 veth 口，實現 namespace 間的 L2 層網路通訊。
	- 相當於模擬多個 Pod 在同一個虛擬 switch 上。
4. iptables + IP forwarding 模擬 NAT 出網
	- 對 NetNS 來說，要出網需要：
		- Host 支援 IP forwarding。
		- `iptables` 設定 POSTROUTING（SNAT/MASQUERADE）。
		- Namespace 設定 default route。
	- 這些步驟實際對應到 Pod 出網的 CNI 流程。

| 元素              | 對應物理世界        | 重點概念                         |
| --------------- | ------------- | ---------------------------- |
| Namespace       | 各自的獨立網路房間     | 每個容器/Pod 的網路空間               |
| veth pair       | 網路線，兩頭各在一房間   | 將 namespace 接到 bridge        |
| bridge          | 集線器/交換器       | Pod 所在的虛擬 switch 網段          |
| iptables + MASQ | NAT 出網機器（路由器） | 模擬 Pod 如何透過 Node 出到 Internet |
| ip route        | GPS 導航地圖      | 調整路由方向決定「去哪條路」               |

## 建立 Network Namespace
```bash
ip netns add red
ip netns add blue

# 檢查已建立的 namespace
ip netns list
```

檢查 Namespace 狀態：
- `ip -n <namespace> addr` = `ip netns exec <namespace> ip addr`
- `ip -n <namespace> link` = `ip netns exec <namespace> ip link`

```bash
ip netns exec red  ip addr    # 查看 red  內的網卡與 IP 狀態
ip netns exec blue ip link    # 查看 blue 內的 interface 狀態
```

## 使用 veth pair 建立兩個 namespace 的連線（建立 Peer 接口）
```bash
# 建立一對虛擬網卡（veth pair），彼此相連，像兩端的網路線。
ip link add veth-red type veth peer name veth-blue

# 一端在 Pod/namespace：將這兩個 veth 接口各自移入 red 與 blue namespace，相當於模擬 Pod 網卡。
ip link set veth-red netns red
ip link set veth-blue netns blue

# 另一端連接 bridge（虛擬交換器）：在兩邊的 veth interface 上設定 IP，且設定在同一個子網（/24），才能互通。
ip -n red addr add 192.168.15.1/24 dev veth-red
ip -n blue addr add 192.168.15.2/24 dev veth-blue

# 啟用 interface，否則不會通訊。
ip -n red link set veth-red up
ip -n blue link set veth-blue up
```

測試連線：
```bash
ip netns exec red ping 192.168.15.2
```

## 使用 Linux Bridge 模擬虛擬交換器（橋接多個 namespace）
當你需要 **讓多個 Network Namespaces 彼此通信或接上外部網路**，就需要在 host 上建立一個 **虛擬交換器（virtual switch）**，最常見的方式是使用：
- Linux Bridge（常用於 Docker）
- Open vSwitch（進階 SDN 應用）

以下會以 **Linux Bridge** 為主，教你如何透過 ip 指令 **建立內部虛擬網路**，管理多個 netns。
| 名稱                    | 功能說明                      |
| --------------------- | ------------------------- |
| `Network Namespace`   | 一個隔離的網路環境                 |
| `veth pair`           | 虛擬網卡，一端連 netns、一端連 bridge |
| `Bridge`              | 虛擬交換器，連接不同的 veth 口        |
| `ip route`            | 顯示 routing table（路由表）     |
| `arp -n` 或 `ip neigh` | 顯示 ARP table（IP 對應 MAC）   |

```bash
# 建立 Bridge (虛擬橋接器)
ip link add v-net-0 type bridge

ip link set v-net-0 up
```
> `ip link set <interface> up`
> = `ip link set dev <interface> up`
> = `ip link set device <interface> up`
>
> `v-net-0`: 內部橋接 red/blue 用

重新建立 veth pair 並橋接至 bridge：
```bash
# 先刪掉原本 red 的 veth-red (舊 veth)
ip -n red link del veth-red

# 新建 veth pair
ip link add veth-red type veth peer name veth-red-br
ip link add veth-blue type veth peer name veth-blue-br

# 將 veth 一端放入 namespace
ip link set veth-red netns red
ip link set veth-blue netns blue

# 另一端接到 bridge
ip link set veth-red-br master v-net-0
ip link set veth-blue-br master v-net-0
ip link set veth-red-br up
ip link set veth-blue-br up
```

在 namespace 中設定 IP 與啟動：
```bash
ip -n red addr add 192.168.15.1/24 dev veth-red
ip -n blue addr add 192.168.15.2/24 dev veth-blue

ip -n red link set veth-red up
ip -n blue link set veth-blue up
```

## 與 Host 通訊 + 模擬 NAT 出網（模擬 Internet）
1. 設定 Host 端 IP：
	```bash
	ip addr add 192.168.15.5/24 dev v-net-0
	ping 192.168.15.1  # 測試是否能通 red
	```
2. 新增 NAT 規則（允許 namespace 出網，使用 iptables）：
	```bash
	iptables -t nat -A POSTROUTING -s 192.168.15.0/24 -j MASQUERADE
	```
	```bash
	ip netns exec blue ping 192.168.1.3  # success
	```
	> `ip_forward` （host 端）必須打開，否則無法從 namespace 出網
	> `# 確保 Host 可以進行 IP forwarding（必要）`
	> `echo 1 > /proc/sys/net/ipv4/ip_forward`
3. 在 namespace 中設定 default gateway：
	```bash
	ip netns exec blue ip route add default via 192.168.15.5
	```
4. 測試連線：
	```bash
	ip netns exec blue ping 8.8.8.8
	```

## 網路診斷與排錯工具
```bash
arp
ip netns exec red arp
ip netns exec blue arp

ip netns exec red route
ip netns exec blue route

ip netns exec red iptables -L -n -v
iptables -nvL -t nat

ip netns exec red ss -tnlp
```

## 常見錯誤與排錯提示
| 問題          | 原因與解法                               |
| ----------- | ----------------------------------- |
| `ping` 不通   | interface 沒有 `up`、IP 設定錯（忘記加 `/24`） |
| 無法出網        | 忘記設定 `iptables POSTROUTING` 規則      |
| DNS 查不到     | 沒設 default gateway、沒設定 resolv.conf  |
| Bridge 沒有啟用 (up) | 忘了執行 `ip link set v-net-0 up`       |

## CKA 考試應用場景
| CKA 主題           | Network Namespace 可模擬場景      |
| ---------------- | ---------------------------- |
| CNI Plugin 原理    | Pod-to-Pod 網路結構與虛擬網卡關係       |
| DNS / 網路不通排查     | 查看 route / iptables / ARP 狀態 |
| kube-proxy 行為模擬  | DNAT / MASQUERADE 行為         |
| NetworkPolicy 測試 | 多 namespace 流量隔離測試           |

## 進階補充：DNAT、靜態路由
```bash
# 模擬 DNAT（port forwarding）
iptables -t nat -A PREROUTING -p tcp --dport 80 -j DNAT --to-destination 192.168.15.2:80

# 靜態路由
ip netns exec blue ip route add 192.168.1.0/24 via 192.168.15.5
```

## Docker 與 Namespace 的關係
每個 Docker Container 都使用獨立的 Network Namespace，可用下列方式觀察：
```bash
docker run -it alpine sh

ps aux          # 查看 process
ip link         # 查看 interface 狀態
```

---
# Pre-Requisites – Networking in Docker

## Docker Networking Modes

### `--network none`
- 沒有網路介面，僅有 loopback (lo)。
- 沒有對外通訊能力。
- 用於測試網路隔離或安全環境。

```bash
docker run --network none nginx
```

### `--network host`
- 容器與主機共用網路堆疊。
- 沒有容器獨立的 IP，與主機共用，亦無 NAT。
- 有效省略 port mapping，但多容器會有 port 衝突風險。
	- 例如多個 Nginx 無法同時用 80 port。

```bash
docker run --network host nginx
```

### `--network bridge`（預設模式）
- 使用 Docker 預設的 bridge network (`docker0`)。
- 容器有獨立 IP（如 172.17.0.x）。
- 支援 NAT，容器可透過 container name 互通。
- Docker 提供內建 DNS 做 name resolution。

```bash
# 預設使用 bridge 模式
docker run nginx
```

## Docker Network 檢查指令
```bash
# 查看所有 Docker 建立的網路（bridge、host、none...）
docker network ls

# 查看容器網路詳細資訊，可以觀察容器 IP、bridge 名稱、Port Mapping。
docker inspect <container-id>
```

### Linux Network Namespace 與 Bridge
Docker 背後透過 NetNS、veth pair、Bridge 建構網路。
```bash
# 查看主機的網路介面
ip link
ip addr

# 查看主機中的 NetNS（需手動建立，Docker 不會自動列出）
ip netns

# 手動建立 Linux bridge（如 docker0）
ip link add docker0 type bridge
```


### iptables + NAT（Docker Port Mapping 原理）
Docker 會透過 iptables NAT 表做 DNAT，將主機 port 映射至容器(Port Mapping)：
```bash
# 模擬 Docker port forwarding 行為，將主機的 8080 導向容器內的 80：
iptables -t nat -A PREROUTING -p tcp --dport 8080 \
  -j DNAT --to-destination 172.17.0.2:80
```

範例：
```bash
# 對外開放 port，實際是建立一條 DNAT 規則
docker run -p 8080:80 nginx
```

### Docker 三種網路模式總整理
| 模式           | 是否有獨立 IP | 是否 NAT | 是否能跨容器存取 | 特點說明                        |
| ------------ | -------- | ------ | -------- | --------------------------- |
| `none`       | ❌        | ❌      | ❌        | 完全無網路，僅有 loopback           |
| `host`       | ❌        | ❌      | ✅（共用主機）  | 效率高但無隔離，易衝突                 |
| `bridge`（預設） | ✅        | ✅      | ✅（同網段）   | 最常見，Docker 會設 bridge 與 veth |

---
# Pre-Requisites – Container Networking Interface (CNI)

## 什麼是 CNI？
- CNI（Container Network Interface）是一套容器網路管理的標準接口，常用於 Kubernetes 和 Podman 等容器平台。
- **Kubernetes 本身不處理 Pod 網路**，而是呼叫 CNI Plugin 來完成：
	- IP 分配
	- 建立 veth pair
	- 加入 Bridge
	- 設定路由與 NAT
- 常見的 CNI Plugin 有：**Calico、Flannel、Cilium、Weave Net** 等。


## 手動模擬 CNI Plugin 行為（使用 Docker）
Kubernetes 透過 CNI plugin 背後其實是以下這些低階操作。我們可以透過 Docker 容器手動模擬。
CNI 是 Kubernetes 與 Podman 等容器平台常用的網路管理插件架構。

1. 步驟 1：建立沒有網路的容器
	- 模擬 Pod 剛建立、尚未連上網路
		```bash
		docker run --network=none nginx
		```
2. 步驟 2：取得容器的 NetNS
	```bash
	PID=$(docker inspect -f '{{.State.Pid}}' mynginx)
	mkdir -p /var/run/netns
	ln -sf /proc/$PID/ns/net /var/run/netns/$PID
	```
3. 步驟 3：建立 veth pair
	```bash
	ip link add veth-host type veth peer name veth-container
	```
4. 步驟 4：將容器端加入 NetNS
	```bash
	ip link set veth-container netns $PID
	```
5. 步驟 5：設定 IP 與啟用介面
	- **Host 端**
		```bash
		ip addr add 10.1.1.1/24 dev veth-host
		ip link set veth-host up
		```
	- **Container 端**
		```bash
		ip netns exec $PID ip addr add 10.1.1.2/24 dev veth-container
		ip netns exec $PID ip link set veth-container up
		ip netns exec $PID ip link set lo up
		```
6. 步驟 6：測試連線
	```bash
	ping 10.1.1.2                      # Host ping Container
	ip netns exec $PID ping 10.1.1.1   # Container ping Host
	```

## 小提醒與補充
| 平台         | 網路方式                  | 說明                              |
| ---------- | --------------------- | ------------------------------- |
| Docker     | bridge + iptables NAT | 容器預設透過 NAT 出網                   |
| Kubernetes | CNI + Routable IP     | 每個 Pod 有 routable IP，**不經 NAT** |

- Kubernetes Pod-to-Pod 通訊在同一網段內不經 NAT。
- CNI Plugin 的動作就是：建立 veth pair、加進 NetNS、加入 Bridge、設 IP 與 Route。
