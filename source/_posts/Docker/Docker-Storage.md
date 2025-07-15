---
title: Docker Storage
description: 'Docker 中的儲存架構與常見儲存技術，包含 Volume、Bind Mount、Storage Driver 與 CSI 介面簡介'
date: 2025-07-15 08:26:52
categories: Docker
tags:
- Docker
- Storage
- Volume
---

# Docker Storage Architecture（儲存架構）

## Layered Architecture (分層架構)
每個 Docker 容器都是由唯讀映像層（image layer）和一個可寫容器層（container layer）組成。

- Image Layers
	- Read Only
- Container Layers
	- Read Write
	- Copy-on-Write（寫入時複製）
		> 當容器對檔案進行修改時，Docker 並不會直接修改映像層(唯讀層)的檔案，而是複製該檔案到容器層(可寫層)再修改，這稱為 Copy-on-Write。
		> 這樣可保持映像層不變，保證容器隔離性與一致性。

---
# Storage Drivers
Docker 使用不同的儲存驅動來管理容器層的檔案系統。
> Overlay2 是目前最建議使用的主流驅動（除非有特別需求）。

不同的作業系統與需求會對應不同的驅動：
- **AUFS**（Ubuntu 常見，支援多層）
- **Overlay / Overlay2**（現代 Linux 發行版的預設）
- **ZFS**（支援壓縮與快照）
- **Btrfs**（支援子卷與快照）
- **Device Mapper**（早期 CentOS / RHEL 使用）
- **VFS**（開發/除錯用途，不使用 Copy-on-Write）

Overlay2 建議搭配 xfs 或 ext4 檔案系統，並支援 SELinux。
可以使用以下指令確認目前使用的 Storage Driver：
```bash
docker info | grep Storage
```

---
# Volume Drivers
Volume 是容器外部的持久化儲存方式，支援許多第三方儲存解決方案：
- **local**（預設、儲存在本地 `/var/lib/docker/volumes`）
- **Azure File Storage**
- **DigitalOcean Block Storage**
- **gce-docker**（Google Cloud）
- **GlusterFS**
- **NetApp**
- **Portworx**
- **RexRay**（支援 AWS EBS、EMC 等）
- **Convoy**
- **Flocker**
- **VMware vSphere Storage**

## 使用 Volume Driver 指令範例
```bash
docker run -it \
  --name mysql \
  --volume-driver rexray/ebs \
  --mount src=ebs-vol,target=/var/lib/mysql \
  mysql
```

## Volume Driver 實務補充
如何列出與移除 volume
```bash
docker volume ls
docker volume inspect data_volume
docker volume rm data_volume
```

---
# Docker Volumes（資料卷）

## Volume Mounting（資料卷掛載）
儲存於 `/var/lib/docker/volumes` 內部目錄，由 Docker 管理。
```bash
docker volume create data_volume
docker run -v data_volume:/var/lib/mysql mysql
```

或用另一個 volume：
```bash
docker run -v data_volume2:/var/lib/mysql mysql
```

## Bind Mounting（目錄綁定掛載）
綁定主機檔案系統的特定目錄 (e.g. `/data`)，適合需要直接存取主機檔案的場景。
> ⚠️ 使用 Bind Mount 時，要特別小心主機路徑的權限與 SELinux/AppArmor 安全設定，以免造成容器存取主機敏感檔案。
```bash
docker run -v /data/mysql:/var/lib/mysql mysql
```

或使用 `--mount`（較新且推薦）語法：👍
```bash
docker run --mount type=bind,source=/data/mysql,target=/var/lib/mysql mysql
```

## 差異比較
| 項目      | Volume Mount     | Bind Mount        |
| -------- | ---------------- | ------------------ |
| 定義方式  | 由 Docker 管理    | 綁定主機目錄          |
| 可攜性    | 高               | 低                  |
| 安全性    | 高（Docker 控制） | 較低（直接操作主機，主機目錄直接存取） |
| 備份與復原 | 容易             | 需自定義腳本/工具      |
| SELinux 相容性 | 有良好支援    | 須手動調整對應標籤    |

---
# Container Storage Interface (CSI)
Container Storage Interface 是 K8s 等容器編排系統與儲存解決方案之間的標準接口協定，讓各種儲存供應商能以一致方式整合。

```
      App (Pod)
	  |
	  v
     [ Kubelet ]
	  |
	  v
    [ CSI Plugin ]
	  |
	  v
[ Storage Provider (e.g. AWS EBS, Ceph) ]
```

## CSI 對應的容器接口
- CRI (Container Runtime Interface)：容器運行時介面，對接 Docker、containerd、CRI-O 等。
- CNI (Container Network Interface)：容器網路介面，對接 Flannel、Cilium、Calico 等。
- CSI (Container Storage Interface)：容器儲存介面，對接各種儲存供應商。
	- 常見支援 CSI 的儲存方案：
		- AWS EBS
		- Azure Disk
		- Google Persistent Disk
		- Dell EMC
		- Portworx
		- GlusterFS
		- Ceph

## CSI 功能要求
| 功能                | RPC 名稱                    | 說明                        | 執行端        |
| ------------------ | --------------------------- | -------------------------- | ------------ |
| 建立 Volume         | `CreateVolume`              | 建立新的儲存資源（PV 對應）    | Controller   |
| 刪除 Volume         | `DeleteVolume`              | 刪除儲存資源                 | Controller   |
| 附加 Volume 到節點   | `ControllerPublishVolume`   | 將 volume attach 至 node   | Controller   |
| 從節點移除 Volume    | `ControllerUnpublishVolume` | detach volume from node   | Controller   |
| 掛載 Volume 給容器使用 | `NodePublishVolume`        | 將 volume 掛載到節點目錄供 Pod 使用 | Node   |
| 卸載 Volume         | `NodeUnpublishVolume`       | 將 volume 從節點目錄卸載     | Node         |

## CSI 底層運作協定
- 使用 gRPC 與容器平台通訊
	> CSI 使用 gRPC 作為通訊協定，能支援多語言、雙向串流與高效能序列化，適合跨平台的容器架構。
- 定義 RPC 方法：CreateVolume、DeleteVolume、ControllerPublishVolume、NodePublishVolume 等

## 補充 CLI 工具與 YAML 例子（進入 Kubernetes 實作）
```yaml
# PersistentVolumeClaim YAML 例子（K8s）
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mypvc
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: gp2
  resources:
    requests:
      storage: 5Gi
```

```yaml
# PersistentVolume (PV) YAML 例子（K8s）
apiVersion: v1
kind: PersistentVolume
metadata:
  name: mypv
spec:
  capacity:
    storage: 5Gi
  accessModes:
    - ReadWriteOnce  # ReadOnlyMany, ReadWriteOnce, ReadWriteMany
  hostPath:
    path: /data/mysql
```

---
# 總結
| 功能     | 技術                 | 說明             |
| ------ | -------------------- | ---------------- |
| 儲存格式   | Copy-on-Write      | 容器層讀寫原理     |
| 儲存驅動   | Overlay2 等        | 控制容器檔案系統    |
| 資料卷     | Volume / Bind Mount | 持久化儲存        |
| 跨平台儲存  | Volume Driver | 整合外部儲存解決方案     |
| 標準儲存協定 | CSI          | 容器與儲存供應商之間的接口 |

