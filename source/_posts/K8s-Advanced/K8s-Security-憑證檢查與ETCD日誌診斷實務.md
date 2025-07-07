---
title: K8s Security | 憑證檢查與 ETCD 日誌診斷實務
description: '檢查 Kubernetes API Server 憑證細節與 ETCD 日誌的實務操作筆記，適用於 K8s 運維與安全排查'
date: 2025-07-07 09:54:22
categories: Kubernetes
tags:
- Certificate
- openssl
- logs
---

# View Certificate Details

## 檢視 K8s API Server 憑證詳細資訊
```bash
# 查看憑證內容
openssl x509 -in /etc/kubernetes/pki/apiserver.crt -text -noout
```

| 欄位                  | 說明                                             |
| -------------------- | ------------------------------------------------ |
| **Certificate Path** | `/etc/kubernetes/pki/apiserver.crt` 檔案位置      |
| **CN Name**          | `Subject: CN=`，常見為 `kube-apiserver` 或主機名稱  |
| **ALT Names**        | `X509v3 Subject Alternative Name`，列出支援的 DNS/IP |
| **Organization**     | `Subject: O=`，通常是 `system:masters`              |
| **Issuer**           | 憑證簽發者，來自 CA 的 CN                             |
| **Expiration**       | `Not After` 欄位表示的憑證過期時間                    |

## 範例解析欄位（從 openssl 輸出中提取）
1. Subject:
```
Subject: O=system:masters, CN=kube-apiserver
```

2. Subject Alternative Name:
```
X509v3 Subject Alternative Name:
    DNS:kubernetes, DNS:kubernetes.default, IP Address:10.96.0.1, ...
```

3. Validity - Not After:
```
Not After : Jul  5 10:33:24 2025 GMT
```

4. Issuer:
```
Issuer: CN=kubernetes-ca
```

---
# Inspect Service Logs

## 查看 etcd 服務狀態與日誌
```bash
# 使用 journalctl 查看 systemd 啟動日誌
journalctl -u etcd.service -l
journalctl -u etcd.service -l --no-pager
```
> `--no-pager`：可避免分頁輸出。
> `-l`（long/full）：顯示不被截斷的完整日誌內容。

---
# View etcd Logs（依部署方式而定）

## 若 etcd 為 Kubernetes static Pod（預設情況）
```bash
# 靜態 Pod 名稱通常為 etcd-[node-name]，可用以下指令查詢
kubectl get pods -n kube-system -o wide
kubectl logs etcd-[node-name] -n kube-system
```
> 使用 `kubectl get pods -n kube-system` 確認實際 Pod 名稱

## 若 etcd 是以 Docker container 運行
```bash
docker ps -a  # 找出 etcd container ID 或名稱
docker logs [container_id or container_name]
```

---
# 其他實用檢查指令

## 檢查 kube-apiserver 的證書與密鑰是否配對
```bash
openssl x509 -noout -modulus -in /etc/kubernetes/pki/apiserver.crt | openssl md5
openssl rsa -noout -modulus -in /etc/kubernetes/pki/apiserver.key | openssl md5
```
> 兩者輸出應一致，否則代表證書與密鑰不配對。

## 驗證 etcd 通訊加密（如使用自簽憑證）
```bash
openssl s_client -connect 127.0.0.1:2379 \
  -cert /etc/kubernetes/pki/etcd/server.crt \
  -key /etc/kubernetes/pki/etcd/server.key \
  -CAfile /etc/kubernetes/pki/etcd/ca.crt
```