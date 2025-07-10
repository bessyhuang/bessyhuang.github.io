---
title: K8s Security | Service Accounts
description: 'K8s ServiceAccount 概念與 CKA 考題實作教學，涵蓋 token 掛載機制、RBAC 綁定與 v1.24 新版行為'
date: 2025-07-10 20:44:22
categories: Kubernetes
tags:
- CKA
- ServiceAccount
- RBAC
- Security
---

# Service Accounts
## ✅ 背景知識與觀念
- K8s 中的每個 Pod，**預設都會掛載一個 ServiceAccount**（`default`），除非在 pod spec 中另行指定。
- ServiceAccount 是 K8s 為工作負載（如 Pod）提供的身份識別方式，用來與 Kubernetes API Server 通訊。
- 可透過 **RBAC** 為 ServiceAccount 綁定適當的權限（`Role` / `ClusterRole`）。
- 舊版中，**ServiceAccount token 會以 Secret 形式自動掛載**到 `/var/run/secrets/kubernetes.io/serviceaccount/token`，可直接透過此 token 與 API Server 溝通。

{% note info %}
- Pod 與其使用的 ServiceAccount 是一對一綁定，但多個 Pod 可共用同一個 ServiceAccount。
- 舊版（v1.24 前）會自動建立一個 Secret token 並掛載；新版（v1.24+）則需透過 `kubectl create token` 明確建立。
- Token 是以 JWT 格式存在，可用來模擬 API 認證呼叫。
{% endnote %}

## 常用操作指令
查詢目前 namespace 中的所有 ServiceAccount
```bash
kubectl get serviceaccounts
```

建立 ServiceAccount
```bash
kubectl create serviceaccount dashboard-sa
```

檢視 ServiceAccount 詳細資訊（包含關聯 Secret）
```bash
kubectl describe serviceaccount dashboard-sa
```

進階驗證 token 掛載內容
```bash
# 查看 SA 對應的 Secret（僅限 < v1.24）
kubectl describe secret $(kubectl get secret | grep dashboard-sa | awk '{print $1}')

# 查看 Pod 掛載的 SA token 路徑
kubectl describe pod my-kubernetes-dashboard

# 驗證 token 是否正確掛載
kubectl exec -it my-kubernetes-dashboard -- ls /var/run/secrets/kubernetes.io/serviceaccount
kubectl exec -it my-kubernetes-dashboard -- cat /var/run/secrets/kubernetes.io/serviceaccount/token
# ca.crt  namespace   token
```

## 常用 YAML
### Pod 指定 ServiceAccount 與驗證 token 掛載
✅ 指定 ServiceAccount 給 Pod
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-kubernetes-dashboard
spec:
  containers:
    - name: dashboard
      image: my-kubernetes-dashboard
  serviceAccountName: dashboard-sa
```

❌ 若不想自動掛載 token
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-kubernetes-dashboard
spec:
  containers:
    - name: dashboard
      image: my-kubernetes-dashboard
  automountServiceAccountToken: false  # 用來停用自動掛載 SA token（常用於強化安全性）
```

✅ 驗證 token 是否掛載
```bash
kubectl exec -it my-kubernetes-dashboard -- ls /var/run/secrets/kubernetes.io/serviceaccount
kubectl exec -it my-kubernetes-dashboard -- cat /var/run/secrets/kubernetes.io/serviceaccount/token
```

### ✅ 與 RBAC 結合：授權 ServiceAccount 權限
YAML 範例：為特定 ServiceAccount 綁定 ClusterRole
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: dashboard-sa-binding
subjects:
  - kind: ServiceAccount
    name: dashboard-sa
    namespace: default
roleRef:
  kind: ClusterRole
  name: view
  apiGroup: rbac.authorization.k8s.io
```

---
# ServiceAccount vs User Account
K8s 有兩種 account。
| 類型             | 使用對象                          | 身份用途                        | 建立方式          |
|------------------|----------------------------------|-------------------------------|-------------------|
| User Account      | 實體使用者（如 admin、developer）   | 與外部系統整合（如 OIDC 認證）       | 不由 K8s 管理      |
| Service Account   | 工作負載（如 Pod、CronJob、Prometheus、Jenkins 等）     | 提供內部 API 認證身份             | 由 K8s 自動建立與管理 |

---
# v1.22 vs. v1.24

## 🔐 v1.22+ ServiceAccount Token 安全性改進
KEP-1205: Bound Service Account Tokens
- 預設 token 為短期有效（time-bound）
- token 綁定於特定 audience（audience-bound）
- 不能被移作他用（object-bound）

```bash
kubectl get pod my-kubernetes-dashboard -o yaml
```

你可以在 Pod Spec 中看到
```bash
serviceAccountToken:
  expirationSeconds: 3600
```

## 🔐 v1.24+ 減少 Secret-based Token 的使用
KEP-2799: ServiceAccount 不再自動產生 Secret-based Token
Kubernetes v1.24 起，ServiceAccount 預設不再自動建立 Secret-based token，取而代之的是：
- 使用 **TokenRequest API** 動態取得短期有效的 Token
- Secret 型 token 僅在需要長期掛載時手動建立
- 減少長效憑證洩漏風險

✅ 動態建立短期有效 token（推薦方式）
```bash
kubectl create token dashboard-sa
```
> 預設有效期限為 1 小時（可使用 `--duration=2h` 調整）

✅ 若仍需手動建立 Secret-based Token（不推薦，但必要時可用）
```bash
apiVersion: v1
kind: Secret
type: kubernetes.io/service-account-token
metadata:
  name: mysecretname
  annotations:
    kubernetes.io/service-account.name: dashboard-sa
```
> ⚠️ 建立後 Kubernetes 會自動填入 token 與 ca.crt，但這個方式只適用於特定場景（如某些外部整合），不建議一般使用者使用。

{% note info %}
v1.22 與 v1.24 差異補充說明
| Kubernetes 版本 | Token 行為                                                 |
| ------------- | -------------------------------------------------------- |
| v1.21 以下      | 預設建立長效 Secret token 並掛載                                  |
| v1.22         | 引入 Bound Token，可控制存活時間 / 使用對象                            |
| v1.24 起       | 預設 **不再建立 Secret token**，必須用 `kubectl create token` 動態請求 |

ServiceAccount Token 使用流程（文字流程圖）
1. Pod 啟動後 → 掛載綁定的 ServiceAccount
2. 若版本 < v1.24 → 自動掛載 Secret Token
3. 若版本 >= v1.24 → 建議用 `kubectl create token` 動態取得
4. 使用 Token → 向 API Server 發送帶有 Bearer Token 的請求
{% endnote %}

## 快速產生測試角色與綁定
```bash
# 建立允許讀取 Pod 的角色
kubectl create role pod-reader \
  --verb=get,list,watch \
  --resource=pods \
  --namespace=default

# 綁定角色給 serviceaccount
kubectl create rolebinding pod-reader-binding \
  --role=pod-reader \
  --serviceaccount=default:dashboard-sa \
  --namespace=default
```
> 用這個方式可快速在 CKA 考試中為特定 ServiceAccount 授權對資源（如 pods）的讀取權限，省下寫 YAML 的時間。

## 實務範例：手動使用 Token 呼叫 API Server
```bash
curl https://<api-server>:6443/api \
  --insecure \
  --header "Authorization: Bearer <YOUR_TOKEN>"
```

---
# CKA 怎麼考？
題目通常給你一個情境，例如：
- 建立一個 ServiceAccount
- 建立 Role / ClusterRole（ex: 讀取某資源）
- 建立對應的 Binding
- 建立 Pod 並使用該 ServiceAccount
- 驗證是否能成功使用 API Token 存取資源

重點技巧：
- 快速使用 `kubectl create` 建立 SA / Role / RoleBinding
- 熟悉修改 Pod yaml 並套用
- 驗證 token 掛載與權限（可用 `curl` + `Bearer`）

---
# 小結
- ServiceAccount 是工作負載與 API Server 通訊的核心機制
- 建議搭配 RBAC 精確授權
- 注意 token 掛載行為在 v1.22 / v1.24 有重大變更
- CKA 考題會實作 SA + RBAC + Pod 驗證整合能力
