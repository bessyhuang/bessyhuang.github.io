---
title: K8s Security | CA
description: '透過 Kubernetes Certificate API 實現用戶憑證申請與簽發流程'
date: 2025-07-07 11:10:30
categories: Kubernetes
tags:
  - CA
  - Certificate
  - CSR
  - kube-controller-manager
---

# K8s CA 與 CSR 憑證簽署流程

## 什麼是 CA (Certificate Authority)？
K8s 使用內建的 Certificate Authority（CA）簽發憑證給用戶、元件或服務，透過自動化控制流程來統一管理：
1. 簽署元件憑證（如 API Server、kubelet）
2. 驗證 Client 憑證（RBAC、API 請求）
3. 使用 Certificates API 自動化 CSR (CertificateSigningRequest) 流程

## CSR (CertificateSigningRequest) 操作流程
CSR 的簽署與核准流程，是由 kube-controller-manager 內部的 controller 自動完成。你可以透過 webhook 或 RBAC 控制誰可以自動批准。

{% note info %}
自動化管理證書簽名：
1. Create CertificateSigningRequest Object
2. Reviewing Requests
3. Approve Requests
4. Share Certs to Users
{% endnote %}

### 1. 建立私鑰與 CSR 檔案
```bash
# 產生私鑰
openssl genrsa -out jane.key 2048

# 產生 CSR 憑證請求，這裡指定 CN 名稱為 jane
openssl req -new -key jane.key -subj "/CN=jane" -out jane.csr
```

### 2. 將 CSR 編碼後建立 K8s CSR 資源
```bash
# Encode: base64 編碼 jane.csr
cat jane.csr | base64 | tr -d '\n'
```
> 建立 CSR 資源時，需將上面 Encode 後的 base64 結果貼入 spec.request 欄位。(jane-csr.yaml)
> `tr -d '\n'` 是移除換行符號，確保貼進 YAML 的 spec.request 欄位是一行文字。

範例 YAML：jane-csr.yaml
```yaml
apiVersion: certificates.k8s.io/v1
kind: CertificateSigningRequest
metadata:
  name: jane
spec:
  request: [貼上 base64 編碼後的 CSR]
  signerName: kubernetes.io/kube-apiserver-client
  usages:
    - client auth
```
```bash
kubectl apply -f jane-csr.yaml
```

### 3. 審核與批准 CSR 請求
```bash
# 查看所有 CSR
kubectl get csr

# 查看 jane 的詳細內容
kubectl get csr jane -o yaml
```

若確認無誤後，執行批准指令：
```bash
kubectl certificate approve jane
```
若需要拒絕請求，可用：
```bash
kubectl certificate deny jane
```

### 4. 取得簽署後的憑證
```bash
# 將憑證存出來
kubectl get csr jane -o jsonpath='{.status.certificate}' | base64 --decode > jane.crt
# jane.crt 是由 Kubernetes CA 簽發的 PEM 格式用戶憑證
```
> 匯出的 jane.crt 為 PEM 編碼（base64 格式）之 X.509 憑證，可直接用於 kubeconfig 或 openssl 驗證。

現在你就擁有：
- jane.key（私鑰）
- jane.crt（由 K8s 簽署的用戶端憑證）

---
# Certificates API 與控制器說明

在 Kubernetes controlplane 中，`kube-controller-manager` 是負責執行多種控制器（controller）的核心元件，其中也包含與憑證相關的操作流程。

當簽署 CSR（CertificateSigningRequest）時，Kube Controller Manager 會使用設定好的 CA 憑證（root certificate）與私鑰（private key）對 CSR 進行簽章。

這些憑證簽發流程，實際上是由 Controller Manager 內部的兩個控制器負責處理：

| Controller Name | 功能說明                        |
| --------------- | ------------------------------ |
| `csr-approving` controller | 決定是否核准 CSR      |
| `csr-signing` controller | 使用 CA 憑證與私鑰簽署 CSR |

## 檢查 Controller Manager 憑證簽署參數
這些控制器會使用 Controller Manager 設定檔中指定的 CA 憑證與私鑰：
```bash
cat /etc/kubernetes/manifests/kube-controller-manager.yaml
```

請確認以下參數存在（K8s 使用此 CA 進行簽名）：
```yaml
  --cluster-signing-cert-file=/etc/kubernetes/pki/ca.crt
  --cluster-signing-key-file=/etc/kubernetes/pki/ca.key
```

---
# CSR 編碼與解碼小技巧
## Encode
```bash
cat jane.csr | base64 | tr -d '\n'
```

## Decode
```bash
echo "[Encode後的結果_base64編碼]" | base64 --decode
```

---
# 小結：K8s CSR 流程重點
1. 建立私鑰與 CSR（含 CN 身份）
2. 將 CSR base64 編碼後建立 Kubernetes 資源
3. 管理員審核並批准（可自動或手動）
4. 匯出簽署憑證，供用戶端連線使用
