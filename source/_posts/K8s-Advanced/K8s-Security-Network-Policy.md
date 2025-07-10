---
title: K8s Security | Network Policy
description: '深入解析 K8s 中的 NetworkPolicy，涵蓋 Ingress / Egress 流量控制、Pod 通訊限制與 CKA 實作重點'
date: 2025-07-11 08:38:35
categories: Kubernetes
tags:
- NetworkPolicy
- Ingress
- Egress
---

# Traffic

## Pod Networking 行為（預設）

- <mark>K8s 預設允許 **所有 Pod 之間互相通訊**，無論是否跨 Node</mark>
- 一旦某個 Pod 被任一 `NetworkPolicy` 匹配到，其網路行為將進入預設拒絕模式（deny all not explicitly allowed）
  - 進入「封閉式網路模型」，**未被允許的流量將預設被封鎖**（deny by default）。
- 其他未匹配到的 Pod 則仍然是全開放。
- 若未設定 `NetworkPolicy`，等同於「全開放」。

| 條件                   | 網路行為                      |
| --------------------- | ------------------------- |
| 沒有任何 NetworkPolicy | 所有 Pod 皆可互通               |
| 只對某些 Pod 設定 NetworkPolicy    | 該些 Pod 進入封閉模式，其他 Pod 照常互通 |


## Ingress vs Egress
```yaml
policyTypes:
- Ingress
- Egress
```

| 類型     | 控制方向 | 說明範例                            | 常見用途                                  |
| ------- | ------- | ---------------------------------- | ----------------------------------------- |
| Ingress | 進入流量（控制誰可以進來） | 控制哪些 Pod/來源可以連進來            | 限制資料庫只能接受來自 API 的流量           |
| Egress  | 離開流量（控制能連誰）    | 控制該 Pod 可向哪些目的地/Port 發出連線 | 限制 Pod 只能對內部服務或特定 IP 發出連線 |
> ✅ NetworkPolicy 是從 **Pod 角度出發**，決定「這個 Pod **能被誰連進來**」（Ingress）與「**它能對誰發出連線**」（Egress）。

---
# 🔒 Network Security

## 實作：限制 API Pod 只能連接 DB
{% note info %}
Q：如果不希望 web pod 能直接溝通 db pod，該怎麼做？
A：db pod 設定一個 NetworkPolicy，只允許 API Pod 發送 3306 port 連線給 DB Pod。
- Allow Ingress Traffic from API Pod on port 3306
- Egress 不設定（預設可以互通）

常見誤區
- 許多人會誤以為沒定義 Egress Policy 就無法連外，其實 **預設還是能連**。
- 只有當某個 `policyType: Egress` 存在時，Egress 才會被限制。

Policy 與 Service 沒有直接關聯的提醒
- NetworkPolicy 是作用在 Pod 層，而非 Service；即使 Service 存在，若對應的 Pod 被封鎖，連線仍會失敗。
{% endnote %}

### db-policy.yaml
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: db-policy
  namespace: default
spec:
  podSelector:
    matchLabels:  # 強調 Label 的 match 條件是「完全符合」
      role: db

  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          role: api
    ports:
    - protocol: TCP
      port: 3306
```

```yaml
  podSelector:
    matchLabels:  # 只會匹配 role=db 的 Pod
      role: db
```
> 所有 `role=db` 的 Pod（如 `mysql`）將只能接受 `role=api` 的 Pod 發送連線（且限 port 3306）；其他來源（如 `web`）都會被阻擋。

```yaml
  podSelector:
    matchLabels:  # 僅匹配同時具有 role=db 且 tier=backend 的 Pod（AND 條件）
      role: db
      tier: backend
```
> Pod 有上述兩個 label，也仍會 match 到只指定 `role: db` 的 policy。

### 建立 NetworkPolicy
```bash
kubectl apply -f db-policy.yaml
```

## 驗證 NetworkPolicy 是否生效

### 使用 busybox 建立測試 Pod
```bash
# 建立有 nc 指令的測試 pod
kubectl run testbox --rm -it --image=busybox:1.28 -- /bin/sh

# 使用 nc 測試連線
nc -zv db-service 3306
```

### 嘗試從 role=web 的 Pod 發送連線
```bash
telnet db-service 3306   # 預期會失敗
```

### 從 role=api 的 Pod 嘗試連線
```bash
kubectl exec api-pod -- nc -zv db-service 3306  # ✅ 預期成功
```

## Egress 實作範例（選擇性補充）
若你希望限制 Pod 不得對外部（如網際網路）連線

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: block-egress
spec:
  podSelector:
    matchLabels:
      app: restricted
  policyTypes:
  - Egress
  egress:
  - to: []   # 空代表不允許任何目標。禁止所有 Egress
```
> ⚠️ 要讓 Egress 有效，Kubernetes 必須搭配有支援 Egress 控制的 CNI，如 Calico、Cilium。
> ⚠️ K8s 本身不處理封包轉發，需透過 CNI 插件實現封包過濾。

# 常見小技巧與注意事項
- `podSelector` 只能篩選 同 namespace 的 Pod
- 跨 namespace 的控制需搭配 `namespaceSelector`
  ```yaml
  from:
  - namespaceSelector:
      matchLabels:
        team: backend  # 允許 team=frontend namespace 中的任意 Pod 存取 db
  ```
  > 可搭配 `namespaceSelector + podSelector` 精確控制跨 namespace 流量
  > 補充：Namespace Label 預設不會存在，需手動打上
- 你也可以設定 IP-based 控制（如限制 Egress 到外部 IP）

```yaml
ingress:
- from:
  - podSelector:
      matchLabels:
        role: api
- from:
  - ipBlock:
      cidr: 10.0.0.0/24
      except:
      - 10.0.0.5/32
```
> 多個 from 視為 OR，只要符合其中一條即可。

## 跨 Namespace 控制完整範例（結合 podSelector + namespaceSelector）
```yaml
from:
- namespaceSelector:
    matchLabels:
      team: backend
  podSelector:
    matchLabels:
      role: api
```

## 幫 Namespace 打 Label
```bash
kubectl label namespace backend team=backend
```

## Ingress/Egress 檢查技巧補充
實務與 CKA 考試中，快速排查 ingress/egress 問題
```bash
# 驗證是否套用
kubectl get networkpolicy -A
kubectl describe networkpolicy <policy-name>
```

```bash
kubectl get pod -o wide
```
> 確認 Pod IP 和 Label，搭配 nc, curl 測試端點通不通。

# CKA 考試怎麼考？
CKA 常見題型如下：
☑️ 給你一組 Pod Label，要你限制某些 Pod 的進入流量（Ingress）
☑️ 配置 NetworkPolicy 讓 Pod 只能從特定來源 存取 DB
☑️ 查 Pod 無法連線，要你排查 `NetworkPolicy` 是否限制太嚴
☑️ 選擇性：限制 Pod 不得連外（Egress deny）

```yaml
# Q: 讓 role=frontend 的 Pod 只能連線到 10.10.10.0/24 的某個外部 API
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: frontend-egress
spec:
  podSelector:
    matchLabels:
      role: frontend
  policyTypes:
  - Egress
  egress:
  - to:
    - ipBlock:
        cidr: 10.10.10.0/24
    ports:
    - protocol: TCP
      port: 443
```

重點技巧：
1. 熟悉 `podSelector` 搭配 `matchLabels`
2. 熟練 `kubectl apply -f xxx.yaml`
3. 驗證技巧：`nc`, `telnet`, `curl`, `kubectl exec`

# 小結
- NetworkPolicy 是控制 Pod 間通訊的強大工具，讓你能實作微服務的最小信任原則（Zero Trust）
- 預設情況下 Pod 全開放通訊，設定任一 Policy 後會觸發預設拒絕行為
- 建議從 DB 等敏感服務開始設定防護（Ingress），逐步擴展到 Egress 控制
- 考試重視：YAML 寫法 + 驗證通訊是否成功 + Label 選擇器語法理解