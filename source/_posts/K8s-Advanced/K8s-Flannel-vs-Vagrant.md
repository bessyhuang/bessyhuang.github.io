---
title: K8s | Flannel vs Vagrant
description: 'Flannel 與 Vagrant 在 CKA 考試準備中的用途與差異比較，協助選擇合適的練習環境與學習重點。'
date: 2025-07-29 23:29:04
categories: Kubernetes
tags:
- CKA
- Flannel
- Vagrant
---

在準備 CKA 考試時，許多人會遇到 Flannel 與 Vagrant 這兩個工具，卻不確定它們之間的關係與哪個更重要。

事實上，Flannel 是 Kubernetes 的網路插件，用於實現 Pod 之間的網路通訊；而 Vagrant 是一種虛擬機管理工具，常被用來快速建立本地測試環境。兩者用途完全不同，但在 CKA 準備中各有其價值。

# Flannel vs Vagrant

| 項目       | Flannel                                 | Vagrant                             |
| -------- | --------------------------------------- | ----------------------------------- |
| 類型       | **CNI 插件**（Container Network Interface） | **虛擬機管理工具**                         |
| 作用       | 提供 Pod-to-Pod 網路通訊能力                    | 建立與管理虛擬機環境（通常搭配 VirtualBox、Libvirt） |
| CKA 考試關聯 | ✔ 需理解 Flannel 基本運作                      | ❌ 本身不考，但可用於搭建練習環境                   |
| 使用場景     | K8s 集群的網路解決方案之一                         | 本地模擬練習 Kubernetes 環境                |
| 是否必備     | 建議理解（你可能會遇到使用 Flannel 的集群）              | 選擇性（你也可以用其他工具建集群，如 kind、minikube）   |

## Flannel 是什麼？
- 是一種 CNI 插件，讓 Kubernetes 中的 Pod 之間可以跨主機通訊。
- 常見於一些預設安裝（例如 kubeadm 搭建的簡單集群）。
- 支援多種後端（如 vxlan、host-gw）。
- ✅ CKA 考試會考網路原理，需要知道 Flannel 是什麼、如何部署、如何排錯。

> Flannel：
> 需要了解它在 Kubernetes 網路中的角色與基本配置，屬於 CKA 網路範圍考點。
> 補充：Flannel 並不是唯一的 CNI 插件，考試或實務中也常見如 Calico、Cilium 等方案。

➡️ 建議學習內容：
- Pod-to-Pod 網路怎麼建立
- veth pair、Network Namespace 的概念
- Flannel 如何配置（YAML）、其 DaemonSet 與 ConfigMap 結構

## Vagrant 是什麼？
🔗 [Vagrant 官方網站](https://www.vagrantup.com/)

- 是一個工具，用來快速啟動多台虛擬機（VM）。
- 適合在本機建一個多節點 Kubernetes 測試環境。
- 通常搭配 VirtualBox，但也支援 VMware、Libvirt。

> Vagrant：
> 不是考點本身，但可用來搭建練習用的多節點 K8s 環境，如果你沒主機或不想用 GCP/AWS，Vagrant 是個不錯的選擇。

➡️ 如果你要自己搭一個 kubeadm 測試集群，Vagrant 是不錯的選擇。
- 但如果你已經用：Minikube、kind（Kubernetes in Docker）、Play with Kubernetes，那就可以不用學 Vagrant。
- 若選擇使用 Vagrant，可搭配 kubeadm、VirtualBox 建立最小化 1 控制平面 + 2 工作節點的練習集群。
	- 可進一步練習：node 管理、Pod 調度、CNI 部署、NetworkPolicy 實作等考點。

# CKA 準備建議
| 目標                     | 工具                | 原因                 |
| ---------------------- | ----------------- | ------------------ |
| 學習 CNI、Pod-to-Pod 網路原理 | Flannel（或 Calico） | 真正用在考試情境與實際集群中     |
| 自建練習用多節點測試環境           | Vagrant + kubeadm | 練習環境一套建好就能反覆用來做實作題 |
| 快速模擬環境（單節點）            | kind / minikube   | 輕量，啟動快速，適合初期練習     |

# 結論整理
- **Flannel 是 CKA 網路考點之一**，需理解其作用與部署方式。
- **Vagrant 雖非考點，但適合用來搭建多節點實機環境進行實作練習**。
- 根據個人電腦資源與需求選擇練習環境，輕量化首選 kind，進階可用 Vagrant + kubeadm。
