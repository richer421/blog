---
title: Kubernetes Pod 生命周期详解
date: 2026-01-18 22:49:18
tags:
  - Kubernetes
  - Pod
  - 生命周期
categories:
  - 云原生
  - Kubernetes
---

## 什么是 Pod 生命周期

Pod 生命周期描述了 Pod 从创建到终止的整个过程，包括各个阶段的状态变化、容器的启动和停止、以及相关的生命周期钩子。理解 Pod 生命周期对于排查问题、优化应用部署至关重要。

<!-- more -->

## Pod 的五个阶段（Phase）

Pod 的 `status.phase` 字段表示 Pod 当前所处的阶段：

### 1. Pending（挂起）

**含义**：Pod 已被 Kubernetes 接受，但有一个或多个容器尚未创建运行。

**可能的原因**：
- 正在下载镜像
- 等待调度到节点
- 资源不足，无法调度
- 存储卷挂载失败

**示例场景**：
```bash
# 查看 Pending 状态的原因
kubectl describe pod <pod-name>
```

### 2. Running（运行中）

**含义**：Pod 已经绑定到节点，所有容器都已创建，至少有一个容器正在运行、启动或重启中。

**特点**：
- Pod 已成功调度到节点
- 容器正在执行
- 可以通过 Service 访问

### 3. Succeeded（成功）

**含义**：Pod 中的所有容器都已成功终止，并且不会再重启。

**适用场景**：
- Job 类型的任务
- 批处理作业
- 一次性任务

**示例**：
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: task-pod
spec:
  restartPolicy: Never  # 不重启
  containers:
  - name: task
    image: busybox
    command: ["sh", "-c", "echo 'Task completed' && exit 0"]
```

### 4. Failed（失败）

**含义**：Pod 中的所有容器都已终止，且至少有一个容器以失败状态终止。

**常见原因**：
- 容器退出码非零
- 容器被系统终止
- 镜像拉取失败
- 启动命令错误

### 5. Unknown（未知）

**含义**：无法获取 Pod 状态，通常是因为与 Pod 所在节点通信失败。

**可能原因**：
- 节点网络故障
- 节点宕机
- kubelet 停止工作

## Pod 状态条件（Conditions）

除了 Phase，Pod 还有更详细的状态条件：

```bash
kubectl get pod <pod-name> -o jsonpath='{.status.conditions}'
```

### 主要的 Condition 类型

| 类型 | 说明 |
|------|------|
| PodScheduled | Pod 已被调度到节点 |
| Initialized | 所有 Init 容器已成功完成 |
| ContainersReady | Pod 中所有容器都已就绪 |
| Ready | Pod 可以接收流量 |

## 容器状态

每个容器有三种可能的状态：

### 1. Waiting（等待）

容器正在等待启动，可能在拉取镜像或等待依赖。

```bash
# 查看等待原因
kubectl get pod <pod-name> -o jsonpath='{.status.containerStatuses[0].state.waiting}'
```

### 2. Running（运行中）

容器正在执行，没有问题。

### 3. Terminated（已终止）

容器已完成执行或因某种原因失败。

```bash
# 查看终止原因和退出码
kubectl get pod <pod-name> -o jsonpath='{.status.containerStatuses[0].state.terminated}'
```

## Init 容器

Init 容器在应用容器启动之前运行，用于初始化工作。

**特点**：
- 按顺序执行
- 必须全部成功完成
- 失败会导致 Pod 重启

**示例**：
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: init-demo
spec:
  initContainers:
  - name: init-service
    image: busybox
    command: ['sh', '-c', 'until nslookup myservice; do echo waiting; sleep 2; done']
  containers:
  - name: app
    image: nginx
```

## 容器生命周期钩子

Kubernetes 提供两种生命周期钩子：

### PostStart（启动后）

容器创建后立即执行，但不保证在容器 ENTRYPOINT 之前执行。

**示例**：
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: lifecycle-demo
spec:
  containers:
  - name: nginx
    image: nginx
    lifecycle:
      postStart:
        exec:
          command: ["/bin/sh", "-c", "echo 'Container started' > /usr/share/message"]
```

### PreStop（停止前）

容器终止前执行，用于优雅关闭。

**示例**：
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: lifecycle-demo
spec:
  containers:
  - name: nginx
    image: nginx
    lifecycle:
      preStop:
        exec:
          command: ["/bin/sh", "-c", "nginx -s quit; while killall -0 nginx; do sleep 1; done"]
```

## 容器探针（Probes）

探针用于检测容器的健康状态：

### 1. Liveness Probe（存活探针）

检测容器是否运行，失败则重启容器。

```yaml
livenessProbe:
  httpGet:
    path: /healthz
    port: 8080
  initialDelaySeconds: 3
  periodSeconds: 3
```

### 2. Readiness Probe（就绪探针）

检测容器是否准备好接收流量，失败则从 Service 中移除。

```yaml
readinessProbe:
  httpGet:
    path: /ready
    port: 8080
  initialDelaySeconds: 5
  periodSeconds: 5
```

### 3. Startup Probe（启动探针）

检测容器是否已启动，用于慢启动容器。

```yaml
startupProbe:
  httpGet:
    path: /startup
    port: 8080
  failureThreshold: 30
  periodSeconds: 10
```

## 重启策略（RestartPolicy）

控制容器失败后的重启行为：

| 策略 | 说明 | 适用场景 |
|------|------|----------|
| Always | 总是重启（默认） | 长期运行的服务 |
| OnFailure | 失败时重启 | Job、批处理任务 |
| Never | 从不重启 | 一次性任务 |

**示例**：
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: restart-demo
spec:
  restartPolicy: OnFailure
  containers:
  - name: app
    image: busybox
    command: ["sh", "-c", "exit 1"]
```

## Pod 终止流程

当删除 Pod 时，会经历以下步骤：

1. **用户发送删除请求**
   ```bash
   kubectl delete pod <pod-name>
   ```

2. **Pod 状态变为 Terminating**
   - Pod 从 Service 的 Endpoints 中移除
   - 不再接收新的流量

3. **执行 PreStop 钩子**
   - 如果配置了 PreStop，会先执行
   - 默认超时时间 30 秒

4. **发送 SIGTERM 信号**
   - 容器收到终止信号
   - 应用应该开始优雅关闭

5. **等待宽限期（Grace Period）**
   - 默认 30 秒
   - 可通过 `terminationGracePeriodSeconds` 配置

6. **发送 SIGKILL 信号**
   - 如果超过宽限期仍未终止
   - 强制杀死容器

**配置宽限期示例**：
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: graceful-shutdown
spec:
  terminationGracePeriodSeconds: 60
  containers:
  - name: app
    image: nginx
```

## Pod 生命周期完整流程图

```
创建 Pod
   ↓
Pending (调度、拉取镜像)
   ↓
Init 容器执行
   ↓
PostStart 钩子
   ↓
Running (容器运行)
   ↓
Liveness/Readiness 探针检测
   ↓
收到删除信号
   ↓
PreStop 钩子
   ↓
SIGTERM 信号
   ↓
宽限期等待
   ↓
SIGKILL 信号
   ↓
Succeeded/Failed
```

## 实用调试命令

```bash
# 查看 Pod 状态
kubectl get pod <pod-name> -o wide

# 查看详细信息
kubectl describe pod <pod-name>

# 查看容器日志
kubectl logs <pod-name> -c <container-name>

# 查看前一个容器的日志（重启后）
kubectl logs <pod-name> -c <container-name> --previous

# 查看 Pod 事件
kubectl get events --field-selector involvedObject.name=<pod-name>

# 实时查看 Pod 状态变化
kubectl get pod <pod-name> -w
```

## 最佳实践

1. **合理配置探针**
   - 设置适当的 `initialDelaySeconds` 避免过早检测
   - 区分 Liveness 和 Readiness 的用途

2. **优雅关闭**
   - 实现 PreStop 钩子或处理 SIGTERM 信号
   - 设置足够的 `terminationGracePeriodSeconds`

3. **使用 Init 容器**
   - 分离初始化逻辑
   - 确保依赖服务就绪

4. **监控 Pod 状态**
   - 关注 Pod 的 Conditions
   - 及时处理异常状态

## 总结

- Pod 生命周期包括 Pending、Running、Succeeded、Failed、Unknown 五个阶段
- Init 容器用于初始化，必须在应用容器前成功完成
- 生命周期钩子（PostStart/PreStop）用于容器启动和关闭时的特殊处理
- 探针（Liveness/Readiness/Startup）用于健康检查
- 理解 Pod 终止流程有助于实现优雅关闭

## 参考资料

- [Kubernetes 官方文档 - Pod Lifecycle](https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/)
- [Container Lifecycle Hooks](https://kubernetes.io/docs/concepts/containers/container-lifecycle-hooks/)
- [Configure Liveness, Readiness and Startup Probes](https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/)
