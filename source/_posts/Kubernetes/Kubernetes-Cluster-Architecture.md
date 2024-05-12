---
title: Kubernetes | Cluster Architecture
description: ' '
date: 2024-04-16 23:21:59
categories: Kubernetes
tags:
- Concepts
- 觀念解說
---

`Kubernetes`
- 目的是讓容器（Container）的管理變得容易、允許被監控且自動化。

`Node`
- `指的是 Worker Nodes 的部分，不是 Master！`
- A node is a worker machine in Kubernetes. A worker node may be a VM or physical machine, depending on the cluster. 
- It has local daemons or services necessary to run Pods and is managed by the control plane. The daemons on a node include kubelet, kube-proxy, and a container runtime implementing the CRI such as Docker.

`Container`
- 輕量級的容器，打包所有必要且相依的套件、程式碼等，提供應用程式一致的運行環境，使應用程式在不同的作業系統上也能夠重建和運行。


# Kubernetes Cluster Architecture

Kubernetes cluster architecture
![](https://kubernetes.io/images/docs/kubernetes-cluster-architecture.svg)

# Kubernetes Components

The components of a Kubernetes cluster
![](https://kubernetes.io/images/docs/components-of-kubernetes.svg)

## <font color=LightCoral>【 Master 】</font>
> aka. 【 Control Plane Components 】
> 職責：`負責監控與管理容器`

<font color=LightCoral>Purpose: Manage, Plan, Schedule, Monitor Nodes</font>


### kube-apiserver
Also known as: API server

The API server is a component of the Kubernetes control plane that exposes the Kubernetes API. The API server is the front end for the Kubernetes control plane.

📌 K8s 最高層次的管理介面，負責協調集群中的所有操作。


### etcd
Consistent and highly-available key value store used as Kubernetes' backing store for all cluster data.

📌 負責儲存集群中不同節點的資訊。


### kube-scheduler
Control plane component that watches for newly created Pods with no assigned node, and selects a node for them to run on.

Factors taken into account for scheduling decisions include: individual and collective resource requirements, hardware/software/policy constraints, affinity and anti-affinity specifications, data locality, inter-workload interference, and deadlines.

📌 負責調度節點上的容器或應用程序。基於容器的資源需求、Worker Node 容量、任何其他策略或約束，來識別正確的節點以放置容器。


### kube-controller-manager
Control plane component that runs controller processes.

There are many different types of controllers. (e.g. Node controller, Job controller, EndpointSlice controller, ServiceAccount controller ...)

📌 不同控制器，處理不同的功能。


## <font color=LightCoral>【 Worker Nodes 】</font>
> aka. 【 Node Components 】
> 職責：`負責裝載容器`

<font color=LightCoral>Purpose: Host Application as Containers</font>


### kubelet
An agent that runs on each node in the cluster. It makes sure that containers are running (and healthy) in a Pod.

📌 相當於船長，是在集群中各個節點（船隻）上運行的代理人，聽從 kube-apiserver 的指令來管理容器。例如：根據需要在節點上部署或銷毀容器。


### kube-proxy
kube-proxy is a network proxy that runs on each node in your cluster, implementing part of the Kubernetes Service concept.

kube-proxy maintains network rules on nodes. These network rules allow network communication to your Pods from network sessions inside or outside of your cluster.

📌 負責集群內不同 worker node 之間的通訊。


### Container runtime
A fundamental component that empowers Kubernetes to run containers effectively. It is responsible for managing the execution and lifecycle of containers within the Kubernetes environment.

Kubernetes supports container runtimes such as containerd, CRI-O, rkt, and any other implementation of the Kubernetes CRI (Container Runtime Interface).

📌 Container 是運行應用程式的獨立環境。


---
# References
- [Cluster Architecture](https://kubernetes.io/docs/concepts/architecture/)
- [Kubernetes Components](https://kubernetes.io/docs/concepts/overview/components/)

