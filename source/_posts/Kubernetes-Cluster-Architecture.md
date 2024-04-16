---
title: Kubernetes | Cluster Architecture
description: ' '
date: 2024-04-16 23:21:59
categories: Kubernetes
tags:
- Concepts
- 觀念解說
---

# Kubernetes Cluster Architecture

Kubernetes cluster architecture
![](https://kubernetes.io/images/docs/kubernetes-cluster-architecture.svg)

# Kubernetes Components

The components of a Kubernetes cluster
![](https://kubernetes.io/images/docs/components-of-kubernetes.svg)

## <font color=LightCoral>【 Master 】</font>
> aka. 【 Control Plane Components 】

<font color=LightCoral>Purpose: Manage, Plan, Schedule, Monitor Nodes</font>


### kube-apiserver
Also known as: API server

The API server is a component of the Kubernetes control plane that exposes the Kubernetes API. The API server is the front end for the Kubernetes control plane.


### etcd
Consistent and highly-available key value store used as Kubernetes' backing store for all cluster data.


### kube-scheduler
Control plane component that watches for newly created Pods with no assigned node, and selects a node for them to run on.

Factors taken into account for scheduling decisions include: individual and collective resource requirements, hardware/software/policy constraints, affinity and anti-affinity specifications, data locality, inter-workload interference, and deadlines.


### kube-controller-manager
Control plane component that runs controller processes.

There are many different types of controllers. (e.g. Node controller, Job controller, EndpointSlice controller, ServiceAccount controller ...)


## <font color=LightCoral>【 Worker Nodes 】</font>
> aka. 【 Node Components 】

<font color=LightCoral>Purpose: Host Application as Containers</font>


### kubelet
An agent that runs on each node in the cluster. It makes sure that containers are running (and healthy) in a Pod.


### kube-proxy
kube-proxy is a network proxy that runs on each node in your cluster, implementing part of the Kubernetes Service concept.

kube-proxy maintains network rules on nodes. These network rules allow network communication to your Pods from network sessions inside or outside of your cluster.


### Container runtime
A fundamental component that empowers Kubernetes to run containers effectively. It is responsible for managing the execution and lifecycle of containers within the Kubernetes environment.

Kubernetes supports container runtimes such as containerd, CRI-O, and any other implementation of the Kubernetes CRI (Container Runtime Interface).


---
# References
- [Cluster Architecture](https://kubernetes.io/docs/concepts/architecture/)
- [Kubernetes Components](https://kubernetes.io/docs/concepts/overview/components/)

