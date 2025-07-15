---
title: K8s Storage
description: 'K8s 中 Volume、PersistentVolume、PersistentVolumeClaim、StorageClass 與動態供應（Dynamic Provisioning）的完整教學'
date: 2025-07-15 10:45:13
categories: Kubernetes
tags:
- PV
- PVC
- StorageClass
---

# Volumes & Mounts
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: random-number-generator
spec:
  containers:
  - image: alpine
    name: alpine
    command: ["/bin/sh", "-c"]
    args: ["shuf -i 0-100 -n 1 >> /opt/number.out; sleep 3600"]
    volumeMounts:
    - mountPath: /opt
      name: data-volume

  volumes:
  - name: data-volume
    hostPath:
      path: /data
      type: Directory
```
- 這個範例定義了一個使用 `hostPath` 類型的 Volume，把主機上的 `/data` 目錄掛載到 Pod 內的 `/opt` 目錄。
- 重點：`hostPath` 是直接使用 Node 上的檔案系統目錄，不具備跨節點共享能力，只適合單節點測試環境。
- 提醒：不建議在 production 環境或多節點集群中使用 `hostPath`，因為 Pod 若重新排程到不同節點，資料無法保留。

## Volume Types

### Volume Types 總覽與用途
K8s 支援多種儲存類型（Volume Types），依據底層儲存設備與需求選擇適合的類型。

| 類型                             | 說明                                         | 適用場景           |
| ------------------------------- | -------------------------------------------- | ---------------- |
| **NFS**                         | 網路檔案系統（Network File System），可在多節點間共享資料       | 多 Pod 共享資料存取     |
| **Ceph (via Rook)**             | 分散式區塊儲存系統，透過 Rook Operator 管理，可動態擴充且高可用      | 企業級高可用儲存與資料共享    |
| **Cloud Provider StorageClass** | AWS EBS、GCP PD、Azure Disk 等透過 CSI 插件支援的動態卷供應 | 雲端原生環境、需持久化的儲存方案 |
| ~~GlusterFS~~                   | 早期分散式儲存方案，功能完備但已不再積極維護，官方建議改用 CSI Plugin     | ❌ 不建議新專案使用       |
| ~~Flocker~~                     | 過時的卷管理工具，已停止維護                               | ❌ 已淘汰            |
| ~~ScaleIO~~                     | Dell EMC 收購後停止支援，從官方文件移除                     | ❌ 已淘汰            |

> 建議重點：
> 專注學習 `StorageClass`、`CSI` 以及 `動態供應（Dynamic Provisioning）`，這是當前 K8s 儲存的主流和考試重點。

### Volume Types 分類概覽
| 類別                 | 代表類型                                                      | 說明                                 |
| ------------------ | --------------------------------------------------------- | ---------------------------------- |
| **暫存型**            | `emptyDir`, `hostPath`, `configMap`, `secret`             | 與 Pod 同生命週期，Pod 刪除後資料消失            |
| **外部型**            | `nfs`, `awsElasticBlockStore`, `gcePersistentDisk`, `csi` | 依賴外部儲存系統，可實現持久化存取                  |
| **Volume Plugins** | `csi`                                                     | 新一代儲存介面標準，統一管理不同存儲系統，取代 FlexVolume |

### 快速判斷何時使用
| 需求                    | 使用建議                                                   |
| --------------------- | ------------------------------------------------------ |
| 短暫資料儲存、不需持久化          | `emptyDir`、`hostPath` 等暫存型 Volume                      |
| Pod 重啟後仍須保留資料         | 使用 PersistentVolume (PV) + PersistentVolumeClaim (PVC) |
| 需要自動創建儲存、簡化管理         | PVC + StorageClass（動態供應）                               |
| 與雲端底層儲存整合（例如 AWS EBS） | StorageClass + CSI 插件                                  |

---
# Persistent Volume (PV)

## 定義與概念
- **PersistentVolume (PV)** 是由 `集群管理員（admin）` 提供和管理的持久化儲存資源。
- PV 的生命週期獨立於 Pod，`Pod 刪除後 PV 依然存在`，資料不會跟著消失。
- PV 可以被多個 Pod 以不同方式重複使用，但要依照其 Access Mode 限制。
- PV 來源可為本地硬碟、網路檔案系統（NFS）、公有雲磁碟（AWS EBS、GCP PD）、Ceph、或其他透過 CSI 插件支持的存儲。

## PV 重要欄位說明
| 欄位                              | 作用                                                                                                           |
| ------------------------------- | ------------------------------------------------------------------------------------------------------------ |
| `accessModes`                   | 存取模式，常見有三種：<br> - ReadWriteOnce (RWO)：單節點可讀寫<br> - ReadOnlyMany (ROX)：多節點只讀<br> - ReadWriteMany (RWX)：多節點可讀寫 |
| `capacity`                      | 儲存容量大小，如 `1Gi`, `10Gi` 等                                                                                     |
| `persistentVolumeReclaimPolicy` | Reclaim Policy (回收策略)，代表當 PVC 刪除後，PV 的處理方式：<br> - `Retain`：保留資料，不刪除 PV （需手動清理底層資源）<br> - `Recycle`：簡單清理（不推薦）<br> - `Delete`：自動刪除 PV 及底層資源（常用於雲端儲存，如 AWS EBS）     |
| `storageClassName`              | 代表此 PV 屬於哪個 StorageClass（可選，常見於動態供應場景）                                                                       |
| `volumeSource`                  | 底層實際存儲類型，如 `awsElasticBlockStore`、`nfs`、`hostPath` 等                                                         |

## 範例 YAML 說明
pv-definition.yaml
```yaml
# PersistentVolume (PV) YAML 例子（K8s）
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-vol1
spec:
  accessModes:
    - ReadWriteOnce  # ReadOnlyMany, ReadWriteOnce, ReadWriteMany
  capacity:
    storage: 1Gi
  persistentVolumeReclaimPolicy: Retain
  awsElasticBlockStore:
    volumeID: <volume-id>
    fsType: ext4
```
- 此 PV 使用 AWS Elastic Block Store（EBS）作為底層儲存。
- Access Mode 是單節點可讀寫 (`ReadWriteOnce`)。
- 回收策略設為 `Retain`，刪除 Pod 或 PVC 後，PV 不會自動刪除。
- `volumeID` 是 EBS 磁碟 ID，需要替換成實際磁碟。

## PV 管理指令
```bash
kubectl create -f pv-definition.yaml  # 建立 PV

kubectl get pv                        # 查詢所有 PV 狀態
kubectl get persistentvolume          # 同上

kubectl describe pv pv-vol1           # 查看特定 PV 詳細資訊
kubectl delete pv pv-vol1             # 刪除 PV

```

## 補充說明
- `PV 不是用戶直接使用的資源`，Pod 是透過 PersistentVolumeClaim (PVC) 向 PV 請求使用權。
- PV 與 PVC 配對時會根據容量與 AccessMode 等條件進行綁定（binding）。
- 在動態供應場景下，多半不需要手動創建 PV，由 StorageClass 自動生成。

---
# Persistent Volume Claim (PVC)

## 定義與概念
- **PersistentVolumeClaim (PVC)** 是開發者或使用者向 K8s 叢集提出的儲存需求申請。
- PVC 就像是一張「我要多少容量、什麼存取模式的儲存空間」的 `申請單`。
- 使用者不需知道底層 PV 的細節，Kubernetes 會自動幫 PVC 找到符合條件的 PV 綁定（Binding）。
- 也可配合 StorageClass 實現 `動態供應（Dynamic Provisioning）`，讓系統自動創建 PV。

## PVC 重要欄位說明
| 欄位                           | 作用                                                                                   |
| ---------------------------- | ------------------------------------------------------------------------------------ |
| `accessModes`                | 請求的存取模式，如：<br>- ReadWriteOnce (RWO)<br>- ReadOnlyMany (ROX)<br>- ReadWriteMany (RWX) |
| `resources.requests.storage` | 請求的儲存容量大小，例如 500Mi、1Gi 等                                                             |
| `storageClassName` (可選)      | 指定 StorageClass 名稱，用於動態供應或匹配特定 PV                                                    |
| `selector` (可選)              | 篩選特定標籤的 PV（較少用）                                                                      |

## PVC 與 PV 綁定條件（Binding）
K8s 在綁定 PVC 和 PV 時，會根據以下條件來匹配：
- **Sufficient Capacity**：PV 必須有足夠的容量大於等於 PVC 請求
- **Access Modes**：PV 的存取模式需包含 PVC 請求的模式
- **Volume Mode**：如 Filesystem 或 Block，需一致
- **StorageClass**：PVC 指定的 StorageClass 與 PV 相符
- **Selector**：若有，符合 PV 的標籤選擇器

## PVC YAML 範例
pvc-definition.yaml
```yaml
# PersistentVolumeClaim YAML 例子（K8s）
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: myclaim
spec:
  accessModes:
    - ReadWriteOnce

  resources:
    requests:
      storage: 500Mi
  # storageClassName: standard  # 可選，指定 StorageClass 名稱
```

## 管理指令
```bash
kubectl create -f pvc-definition.yaml    # 建立 PVC

kubectl get pvc                          # 查詢所有 PVC
kubectl get persistentvolumeclaim        # 同上

kubectl describe pvc <name>              # 查看特定 PVC 詳細資訊
kubectl get sc                           # 查看 StorageClass
kubectl delete pvc <name>                # 刪除 PVC
```

## 補充重點
- **PVC 是使用者的申請，PV 是管理員或系統提供的實體資源**，兩者透過綁定（Binding）產生關聯。
- 在動態供應場景中，PVC 指定 StorageClass，K8s 自動幫你創建並綁定 PV，使用者不用管理 PV 細節。
- 若沒有符合的 PV，PVC 會一直處於 Pending 狀態，等待符合的 PV 被建立或釋放。


---
# PV vs PVC — 儲存解耦核心概念

## 定義與差異
- `PersistentVolume (PV)`：由 admin 事先建立，或透過 StorageClass 自動動態建立的持久化儲存資源。屬於叢集級別，與 Pod 生命週期無關。
- `PersistentVolumeClaim (PVC)`：user 向叢集提出的儲存請求，聲明所需容量、存取模式等條件，Kubernetes 會自動綁定符合條件的 PV。
- ✅ **目標：實現計算（Pod）與儲存（Volume）的解耦。**

```
[ 使用者 → PVC ] ──→ [ Kubernetes 配對符合的 PV ]
                               ↓
                  [ PV 與 PVC 綁定，提供給 Pod 掛載 ]
```
## StorageClass
- 定義 PV 的動態供應方式，包含儲存類型（如 AWS EBS、NFS）、回收策略、參數等。
- 搭配 PVC 使用，能讓 Kubernetes 自動建立並綁定對應 PV，簡化管理流程。

## CSI（Container Storage Interface）
- Kubernetes 官方推薦的統一儲存介面標準，解決傳統 FlexVolume 外掛不易擴充與管理問題。
- 支援各大雲端（AWS、GCP、Azure）與企業級儲存（如 Ceph、VMware）。
- 幾乎所有新版儲存方案皆透過 CSI 實作。

## 動態供應（Dynamic Provisioning）
- 當 PVC 被建立時，若無符合的 PV，Kubernetes 可根據 PVC 指定的 StorageClass 自動建立對應的 PV。
- 無需手動預先創建 PV，提升資源管理彈性與效率。

## 使用流程簡述
1. admin 建立 PV 或設定好支援 CSI 的 StorageClass。
2. user 建立 PVC，聲明所需容量與存取模式。
3. K8s 根據 PVC 條件，尋找並綁定符合的 PV（或動態建立 PV）。
4. Pod 使用 PVC 所聲明的 volume 進行資料讀寫。


---
# 在 Pod / Deployment 中使用 PVC

## ✅ 基本用法：Pod 掛載 PVC
建立 PVC 後，可以在 Pod 中透過 `volumes.persistentVolumeClaim` 搭配 `volumeMounts` 掛載資料卷：
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: mypod
spec:
  containers:
  - name: myfrontend
    image: nginx
    volumeMounts:
    - mountPath: "/var/www/html"
      name: mypd
  volumes:
  - name: mypd
    persistentVolumeClaim:
      claimName: myclaim  # 指向已建立的 PVC 名稱
```

## 📌 在 Deployment 或 ReplicaSet 中掛載 PVC
與 Pod 相同，只是 `volumeMounts` 與 `volumes` 需放入 `spec.template.spec` 中：
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  replicas: 2
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx
        volumeMounts:
        - name: web-content
          mountPath: /usr/share/nginx/html
      volumes:
      - name: web-content
        persistentVolumeClaim:
          claimName: myclaim
```
> ✅ 注意：如果使用的是 ReadWriteOnce 的 PVC，多個 Pod 不可以分布在多個節點上同時掛載。使用 StatefulSet + RWX 或 NFS/CSI 是更合適的選擇。

---
# 📦 StorageClass + 動態供應（Dynamic Provisioning）

## 定義 StorageClass（CKA 考試重點）
當 PVC 沒有對應的 PV 時，K8es 會根據 PVC 所指定的 StorageClass 自動建立 PV，此稱為 `動態供應（Dynamic Provisioning）`。
```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: standard
provisioner: kubernetes.io/aws-ebs  # 或 csi.ebs.amazonaws.com（新版 CSI）
parameters:
  type: gp2                          # AWS EBS 類型
reclaimPolicy: Delete               # PVC 刪除後，自動刪除底層儲存
volumeBindingMode: Immediate        # 立即綁定（或使用 WaitForFirstConsumer）
```

| 欄位                  | 說明                                                   |
| ------------------- | ---------------------------------------------------- |
| `provisioner`       | CSI 插件名稱或內建供應器                                       |
| `parameters`        | 儲存參數，如磁碟類型                                           |
| `reclaimPolicy`     | Delete / Retain 回收策略                                 |
| `volumeBindingMode` | `Immediate`（預設）或 `WaitForFirstConsumer`，後者推薦用於多區雲端部署 |
