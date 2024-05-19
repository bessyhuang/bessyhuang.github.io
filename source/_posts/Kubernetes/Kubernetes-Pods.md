---
title: Kubernetes | Pods
description: ' '
date: 2024-05-14 23:05:37
categories: Kubernetes
tags:
- Concepts
- 觀念解說
---

# Pods

📌 Kubernetes doesn't deploy containers directly on the worker node. The containers are encapsulated into pods.
📌 A pod is a single instance of an application. A pod is the smallest object that you can create in Kubernetes.

- Why does Kubernetes use pods? 
  - The relationship of pods to clusters is why Kubernetes does not run containers directly, instead running pods to ensure that each container within them `shares the same resources and local network`.
  - Grouping containers in this way allows them to communicate between each other as if they shared the same physical hardware, while still remaining isolated to some degree.
  - Same storage, same network, same namespace, same fit as in they will be created together and destroyed together.
  - This is a good thing in the long run because your application is now ready to handle future architectural changes and scale.

## The simplest case

```
        Big        > ----------- > --- > --------- >    Small
Kubernetes Cluster > Worker Node > Pod > Container > Application
```

- A Kubernetes cluster
- A single worker node
- A single pod
- A single container
- A single application

![](Kubernetes-Pods/Kubernetes_Pod.jpg)


## Best Practice

📌 Pods usually have a one-to-one relationship with (same kind) containers running your application.
- `1 Pod : 1 (same kind) Container`
- Multiple containers running in a Pod is a rare use case.

📌 Don't add additional (same kind) containers to an existing pod to scale your application!

📌 當連線到應用程式的使用者增加時，該如何擴展你的應用程式來分攤負載（share the load）？
- `[ X ]` 在同一個 Pod 中，啟動新的 Container。(A cluster, a worker node, two containers in a pod)
- `[ O ]` 新增一個 Pod ，並在其中啟動新的 Container。(A cluster, a worker node, two pods, a container in a pod)
- `[ O ]` 若當前 worker node 沒有足夠容量時，新增一個 worker node，並在該 worker node 上新增一個 Pod，接著在 Pod 上啟動新的 container。(A cluster, two worker nodes, a worker node can have multiple pods)


# 圖解 Kubectl Command

📌 Kubectl Command 會自動創建一個 Pod，並在其內部署一個 Container。


# YAML in Kubernetes




## Pods with YAML



## Service with YAML



## ReplicaSet with YAML



## Deployment with YAML


# References
- [What is a Kubernetes pod?](https://www.redhat.com/en/topics/containers/what-is-kubernetes-pod)
