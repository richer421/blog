---
title: Kubernetes Pod QoS 详解
date: 2026-01-18 19:55:11
tags:
  - Kubernetes
  - Pod
  - QoS
categories:
  - 云原生
  - Kubernetes
---

## 什么是 Pod QoS

QoS（Quality of Service，服务质量）是 Kubernetes 用来管理 Pod 资源分配和驱逐优先级的机制。当节点资源不足时，Kubernetes 会根据 Pod 的 QoS 等级来决定驱逐哪些 Pod，以保证集群的稳定运行。

<!-- more -->

## QoS 的三种等级

Kubernetes 根据 Pod 中容器的资源请求（requests）和限制（limits）配置，自动为 Pod 分配以下三种 QoS 等级：

### 1. Guaranteed（保证级别）

**特点**：
- 最高优先级
- 资源得到完全保证
- 最后被驱逐

**分类条件**：
- Pod 中的每个容器都必须设置 CPU 和内存的 requests 和 limits
- 每个容器的 requests 必须等于 limits

**示例配置**：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: qos-guaranteed
spec:
  containers:
  - name: nginx
    image: nginx
    resources:
      requests:
        memory: "200Mi"
        cpu: "500m"
      limits:
        memory: "200Mi"
        cpu: "500m"
```

### 2. Burstable（突发级别）

**特点**：
- 中等优先级
- 可以使用超过 requests 的资源，但不能超过 limits
- 在 BestEffort 之后被驱逐

**分类条件**：
- Pod 不满足 Guaranteed 的条件
- Pod 中至少有一个容器设置了 CPU 或内存的 requests

**示例配置**：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: qos-burstable
spec:
  containers:
  - name: nginx
    image: nginx
    resources:
      requests:
        memory: "100Mi"
        cpu: "250m"
      limits:
        memory: "200Mi"
        cpu: "500m"
```

### 3. BestEffort（尽力而为级别）

**特点**：
- 最低优先级
- 不保证资源
- 最先被驱逐

**分类条件**：
- Pod 中的所有容器都没有设置 CPU 和内存的 requests 和 limits

**示例配置**：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: qos-besteffort
spec:
  containers:
  - name: nginx
    image: nginx
```

## QoS 等级的判定流程

```
是否所有容器都设置了 requests 和 limits，且 requests == limits？
├─ 是 → Guaranteed
└─ 否 → 是否至少有一个容器设置了 requests？
         ├─ 是 → Burstable
         └─ 否 → BestEffort
```

## 资源驱逐顺序

当节点资源不足时，Kubernetes 按照以下顺序驱逐 Pod：

1. **BestEffort** Pod 最先被驱逐
2. **Burstable** Pod 其次被驱逐（优先驱逐资源使用超过 requests 最多的）
3. **Guaranteed** Pod 最后被驱逐（只有在系统组件需要资源时才会被驱逐）

## 实际应用建议

### 生产环境核心服务
使用 **Guaranteed** 级别，确保关键服务的稳定性：
```yaml
resources:
  requests:
    memory: "2Gi"
    cpu: "1000m"
  limits:
    memory: "2Gi"
    cpu: "1000m"
```

### 一般业务服务
使用 **Burstable** 级别，平衡资源利用率和稳定性：
```yaml
resources:
  requests:
    memory: "512Mi"
    cpu: "500m"
  limits:
    memory: "1Gi"
    cpu: "1000m"
```

### 批处理任务
可以使用 **BestEffort** 级别，充分利用空闲资源：
```yaml
# 不设置 resources
```

## 查看 Pod 的 QoS 等级

使用以下命令查看 Pod 的 QoS 等级：

```bash
kubectl get pod <pod-name> -o jsonpath='{.status.qosClass}'
```

或者查看详细信息：

```bash
kubectl describe pod <pod-name> | grep QoS
```

## 总结

- QoS 是 Kubernetes 资源管理的重要机制
- 合理设置 requests 和 limits 可以提高集群资源利用率
- 生产环境建议为核心服务设置 Guaranteed 级别
- 理解 QoS 机制有助于优化应用的稳定性和性能

## 参考资料

- [Kubernetes 官方文档 - Configure Quality of Service for Pods](https://kubernetes.io/docs/tasks/configure-pod-container/quality-service-pod/)
- [Resource Management for Pods and Containers](https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/)
