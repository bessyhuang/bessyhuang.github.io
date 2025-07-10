---
title: K8s Security | Image Security & SecurityContext
description: 'K8s Image 安全性、Security Context 與 Docker 安全模型介紹，對應 CKA 考點與實務操作整理'
date: 2025-07-10 22:28:17
categories: Kubernetes
tags:
- CKA
- Docker
- ImageSecurity
- SecurityContext
---

# ✅ 背景知識與觀念

## Image Security 觀念
- 避免使用 `latest` tag → **不可預測、風險高**
- 指定明確版本號（如 `nginx:1.24.2`）
- 儘量選擇：
  - 可信來源（官方、企業內部 registry）
  - 已簽署 / 掃描過的映像檔（搭配 Notary、Cosign）
- 配合 `imagePullPolicy` 控制 image 更新策略
- 可搭配 Admission Controller 實作政策限制
  - 如：Gatekeeper + OPA、ImagePolicyWebhook

## Linux Container 安全模型核心概念
- **Namespaces**：隔離資源（PID、網路、Mount 等）
- **cgroups**：限制資源（CPU、Memory）
- **Capabilities**：限制容器能使用哪些權限
- **seccomp**：限制能呼叫哪些 system calls

> Kubernetes 為了安全性，**不建議容器以 root 身份執行**，應使用 non-root user。

---
# Image Security

## ✅ 使用私有映像檔（Private Image）
登入私有 Registry 並使用 image
```bash
docker login private-registry.io
docker run private-registry.io/apps/internal-app
```

建立可供 K8s 使用的 image pull secret
```bash
kubectl create secret docker-registry regcred \
  --docker-server=private-registry.io \
  --docker-username=registry-user \
  --docker-password=registry-password \
  --docker-email=registry-user@org.com
```

Pod 使用 imagePullSecrets 指定 Secret
```yaml
# nginx-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-pod
spec:
  containers:
    - name: nginx
      image: private-registry.io/apps/internal-app:v1.2.3
      imagePullPolicy: IfNotPresent
  imagePullSecrets:
    - name: regcred
```

## imagePullPolicy 行為比較
`imagePullPolicy` 是 Kubernetes Pod 中 container 的一個欄位，用來控制 Kubelet 什麼時候去拉（pull）image。它在部署與調試階段非常重要，尤其當你使用自建 image 或開發中頻繁更新的 image。

| 值              | 行為說明                                                                |
| -------------- | ------------------------------------------------------------------- |
| `Always`       | **每次 Pod 啟動都會重新拉 image**（不管本地有沒有）<br>通常用於 `:latest` image 或確保更新拉到最新 |
| `IfNotPresent` | **只有當本地沒有對應 image 才會去拉**（節省頻寬與啟動時間）                                 |
| `Never`        | **永遠不會去拉 image**，只使用本地已有的 image，拉不到就會失敗（CI/CD、測試環境有時會用）             |
> 預設值會依據 image 是否使用 `latest` 而變化：  
> - 使用 `:latest` → 預設為 `Always`  
> - 指定其他 tag → 預設為 `IfNotPresent`
> ✅ 為了可預測性，強烈建議永遠明確設定 `imagePullPolicy`。

查看 Pod 使用的 Image
```bash
kubectl describe pod mypod | grep Image
```

---
# Docker Security 實務

## ✅ 使用非 root user
Dockerfile 中加入
```dockerfile
FROM ubuntu
USER 1000
```

手動測試時也可以用
```bash
docker run --user=1001 ubuntu sleep 3600
```

## ✅ 使用 capabilities / 限制權限
添加特權能力（如 MAC_ADMIN）
```bash
docker run --cap-add=MAC_ADMIN ubuntu
```

移除某些能力（如 KILL）
```bash
docker run --cap-drop=KILL ubuntu
```

開啟完全特權模式（⚠️ 極高風險）
```bash
docker run --privileged ubuntu
```
> ✅ 實務中，僅當應用需特殊權限（如操作網路 interface 或系統層設定）時才會加 capability，否則建議盡量精簡

---
# Security Context

## ✅ Pod Level 設定
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: web-pod
spec:
  securityContext:
    runAsUser: 1000
  containers:
    - name: ubuntu
      image: ubuntu
      command: ["sleep", "3600"]
```

## ✅ Container Level 設定
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: web-pod
spec:
  containers:
    - name: ubuntu
      image: ubuntu
      command: ["sleep", "3600"]
      securityContext:
        runAsUser: 1000
        capabilities:
          add: ["MAC_ADMIN"]
```
> <mark>若在 Pod 與 Container 都有設定 `runAsUser`，以 Container 層級為主</mark>

{% note info %}
⚠️ 常見錯誤與實務注意事項
- 若容器使用非 root 使用者，掛載 volume 時需確認目錄權限是否能寫入
  - 例：`/data` 目錄應設為 UID:1000 可寫
- 建議 base image 本身已建立非 root 使用者
  - 例：`node:slim`, `nginx:alpine` 常已內建 user

<br>

```yaml
      securityContext:
        runAsUser: 1000
        runAsGroup: 3000
        fsGroup: 2000  # ✅ 掛載 volume 的群組權限自動對應
```
> `fsGroup` 可協助解決 volume 權限問題，尤其當容器非 root 且需寫入掛載目錄時
{% endnote %}

---
# CKA 考題可能怎麼出？
☑️ 指定明確的 image tag（避免 latest）  
☑️ 設定正確的 `imagePullPolicy`  
☑️ 處理 CrashLoopBackOff（image 拉不到）  
☑️ Pod 運行失敗 → 判斷是否權限錯誤（non-root user 無法寫入）  
☑️ 修改 YAML，設定 `securityContext`、指定 user、capabilities

> ❗Dockerfile 本身不會考，但會用錯誤的 YAML 模擬出問題讓你 debug
> 💡 若 Pod CrashLoopBackOff，可使用 `kubectl logs` 與 `kubectl describe pod` 搭配排查

---
# 小結
- 使用非 root 容器 + 明確 image tag 是安全基本原則
- 搭配 RBAC 與 Admission Controller 強化政策落實
- 熟悉 `securityContext` 在 Pod/Container 不同層級的效果
- CKA 考試更重視**實作錯誤排查與安全配置能力**
