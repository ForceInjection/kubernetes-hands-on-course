# Kubernetes Pod QoS 原理与实现

## 1. 引言

在 Kubernetes 集群中，资源管理是一个核心问题。当节点资源（CPU、内存）充裕时，所有 Pod 都能和谐共存；但当资源紧张甚至枯竭时，Kubernetes 需要一种机制来决定“保谁弃谁”。这种机制就是 **QoS (Quality of Service，服务质量)**。

QoS 类（QoS Class）是 Kubernetes 根据 Pod 的资源请求（Requests）和限制（Limits）配置自动赋予 Pod 的一个标签。这个标签不仅决定了 Pod 的调度优先级，更关键的是，它直接映射到了底层的 Linux Cgroups 配置和 OOM（Out of Memory）Score，从而决定了 Pod 在 CPU 争抢时的权重以及在内存不足时的被杀顺序。

本文将从源码角度深入剖析 Kubernetes QoS 的判定逻辑、Cgroup 资源配置以及 OOM 处理机制。

---

## 2. QoS 等级分类与代码判定

Kubernetes 将 Pod 分为三种 QoS 等级，按优先级从高到低依次为：**Guaranteed** > **Burstable** > **BestEffort**。

这一分类逻辑实现在 `pkg/apis/core/helper/qos/qos.go` 文件中的 `ComputePodQOS` 函数里。

### 2.1 Guaranteed (完全保障型)

**定义**：享有最高优先级的资源保障。
**判定条件**：

1. Pod 中的每个容器（包括 Init Containers 和 App Containers）都设置了 CPU 和 Memory 的 Request 和 Limit。
2. 对于每个容器，CPU 的 Request 等于 Limit。
3. 对于每个容器，Memory 的 Request 等于 Limit。

> 注意：如果只设置 Limit 未设置 Request，Kubernetes 会默认 Request = Limit，因此也符合 Guaranteed 条件。

### 2.2 Burstable (弹性突发型)

**定义**：资源需求有弹性，允许在节点有空闲资源时突破 Request 使用更多资源。
**判定条件**：

1. Pod 不符合 Guaranteed 的条件。
2. Pod 中至少有一个容器设置了 CPU 或 Memory 的 Request 或 Limit。

### 2.3 BestEffort (尽力而为型)

**定义**：没有任何资源保障，通常用于非关键性任务。
**判定条件**：

1. Pod 中的所有容器都没有设置任何 CPU 或 Memory 的 Request 或 Limit。

### 2.4 代码实现分析

以下是 `ComputePodQOS` 函数的核心逻辑摘要：

```go
// pkg/apis/core/helper/qos/qos.go

func ComputePodQOS(pod *core.Pod) core.PodQOSClass {
    requests := core.ResourceList{}
    limits := core.ResourceList{}
    isGuaranteed := true

    // ... 遍历所有容器 (Init 和 App) ...
    for _, container := range allContainers {
        // 处理 Requests
        for name, quantity := range container.Resources.Requests {
            // 累加 Request
            // ...
        }
        // 处理 Limits
        qosLimitsFound := sets.NewString()
        for name, quantity := range container.Resources.Limits {
            // 累加 Limit
            // ...
            qosLimitsFound.Insert(string(name))
        }

        // 如果没有设置所有支持 QoS 的资源(CPU/Memory)的 Limit，则不是 Guaranteed
        if !qosLimitsFound.HasAll(string(core.ResourceMemory), string(core.ResourceCPU)) {
            isGuaranteed = false
        }
    }

    // 如果没有任何 Request 和 Limit，则是 BestEffort
    if len(requests) == 0 && len(limits) == 0 {
        return core.PodQOSBestEffort
    }

    // 检查 Request 是否等于 Limit
    if isGuaranteed {
        for name, req := range requests {
            if lim, exists := limits[name]; !exists || lim.Cmp(req) != 0 {
                isGuaranteed = false
                break
            }
        }
    }

    if isGuaranteed && len(requests) == len(limits) {
        return core.PodQOSGuaranteed
    }
    return core.PodQOSBurstable
}
```

---

## 3. QoS 与 CPU 调度 (Cgroup 配置)

QoS 等级决定了 Pod 在 CPU 资源争抢时的相对权重。Kubernetes 通过设置 Cgroup 的参数来实现这一点。由于 Linux Cgroup v1 和 v2 的差异，实现方式有所不同。

### 3.1 Cgroup v1 实现 (`cpu.shares`)

在 Cgroup v1 中，CPU 权重通过 `cpu.shares` 控制。这是一个相对值，默认值通常是 1024。

**计算公式**：

- **Guaranteed / Burstable**: `cpu.shares = (CPU_Request_MilliCores * 1024) / 1000`
  - 例如：Request = 1 CPU (1000m) -> shares = 1024
  - 例如：Request = 500m -> shares = 512
- **BestEffort**: `cpu.shares = 2` (MinShares)

**代码分析** (`pkg/kubelet/cm/helpers_linux.go`):

```go
// MilliCPUToShares converts the milliCPU to CFS shares.
func MilliCPUToShares(milliCPU int64) uint64 {
    if milliCPU == 0 {
        // Docker converts zero milliCPU to unset, which maps to kernel default
        // for unset: 1024. Return 2 here to really match kernel default for
        // zero milliCPU.
        return MinShares
    }
    // Conceptually (milliCPU / milliCPUToCPU) * sharesPerCPU, but factored to improve rounding.
    shares := (milliCPU * SharesPerCPU) / MilliCPUToCPU
    if shares < MinShares {
        return MinShares
    }
    if shares > MaxShares {
        return MaxShares
    }
    return uint64(shares)
}
```

这意味着，在 CPU 满负荷时，BestEffort Pod 获得的 CPU 时间极少，而 Guaranteed 和 Burstable Pod 则根据其 Request 值按比例分配 CPU。

### 3.2 Cgroup v2 实现 (`cpu.weight`)

Cgroup v2 使用 `cpu.weight` 来控制权重，取值范围是 [1, 10000]，默认值是 100。

**计算公式**：
Kubernetes 需要将 v1 的 `cpu.shares` (2-262144) 映射到 v2 的 `cpu.weight` (1-10000)。

- **公式**: `weight = 1 + ((shares - 2) * 9999) / 262142`
- **BestEffort**: 映射结果为 `1`。

**代码分析** (`pkg/kubelet/cm/cgroup_manager_linux.go`):

```go
// getCPUWeight converts from the range [2, 262144] to [1, 10000]
func getCPUWeight(cpuShares *uint64) uint64 {
    if cpuShares == nil {
        return 0
    }
    if *cpuShares >= 262144 {
        return 10000
    }
    return 1 + ((*cpuShares-2)*9999)/262142
}
```

### 3.3 QoS Cgroup 配置对照表

| QoS 类别       | 条件                | cgroup v1 (cpu.shares)      | cgroup v2 (cpu.weight)         | 说明           |
| :------------- | :------------------ | :-------------------------- | :----------------------------- | :------------- |
| **Guaranteed** | CPU Request = Limit | `(Request_m * 1024) / 1000` | `1 + ((shares-2)*9999)/262142` | 正常按比例分配 |
| **Burstable**  | CPU Request ≠ Limit | `(Request_m * 1024) / 1000` | `1 + ((shares-2)*9999)/262142` | 同上           |
| **Burstable**  | 未设置 CPU Request  | `2` (MinShares)             | `1`                            | 视为最低优先级 |
| **BestEffort** | 无 Request/Limit    | `2` (MinShares)             | `1`                            | 最低优先级     |

### 3.4 关键代码调用路径

了解代码调用路径有助于深入理解配置是如何生效的：

1. **Cgroup v1**:

   ```text
   pkg/kubelet/cm/qos_container_manager_linux.go
      │
      ▼
   UpdateCgroups
      │
      ▼
   cgroupManager.Update (pkg/kubelet/cm/cgroup_manager_linux.go)
      │
      ▼
   toResources (设置 cpu.shares)
      │
      ▼
   libcontainer.Manager.Set
      │
      ▼
   写入 /sys/fs/cgroup/cpu/cpu.shares
   ```

2. **Cgroup v2**:

   ```text
   pkg/kubelet/cm/qos_container_manager_linux.go
      │
      ▼
   UpdateCgroups
      │
      ▼
   cgroupManager.Update (pkg/kubelet/cm/cgroup_manager_linux.go)
      │
      ▼
   toResources (计算 cpu.weight)
      │
      ▼
   libcontainer.Manager.Set
      │
      ▼
   写入 /sys/fs/cgroup/kubepods.slice/.../cpu.weight
   ```

---

## 4. QoS 与 内存管控 (OOM Score Adj)

当节点内存不足时，Linux 内核的 OOM Killer 会被触发。Kubernetes 通过调整每个容器进程的 `/proc/<pid>/oom_score_adj` 值，来指导 OOM Killer 优先杀掉哪些进程。`oom_score_adj` 的范围是 `[-1000, 1000]`，值越大越容易被杀。

### 4.1 OOM Score Adj 设定规则

1. **Guaranteed**: `-997`

   - 非常安全，仅次于 `-999` (Kubelet/Docker 守护进程)。
   - 意味着除非系统级关键组件崩溃，否则 Guaranteed Pod 是最后被杀的。

2. **BestEffort**: `1000`

   - 最大值，意味着它是 OOM Killer 的首选目标。

3. **Burstable**: `min(max(2, 1000 - (1000 * Request / Capacity)), 999)`
   - 这是一个动态计算值，范围通常在 `[2, 999]` 之间。
   - **逻辑**：Pod 申请的内存占节点容量的比例越大，OOM Score Adj 越小（越安全）。
   - **例子**：如果一个 Burstable Pod 申请了节点 50% 的内存，得分为 `1000 - 500 = 500`。如果只申请了 0.1%，得分为 `999`。这保护了那些按需申请大量资源的关键 Burstable Pod，而惩罚那些申请很少但可能突然超用的 Pod。

### 4.2 代码实现分析

核心逻辑位于 `pkg/kubelet/qos/policy.go` 中的 `GetContainerOOMScoreAdjust` 函数：

```go
const (
    guaranteedOOMScoreAdj int = -997
    besteffortOOMScoreAdj int = 1000
)

// GetContainerOOMScoreAdjust returns the amount by which the OOM score of all processes in the
// container should be adjusted.
func GetContainerOOMScoreAdjust(pod *v1.Pod, container *v1.Container, memoryCapacity int64) int {
    // 1. 如果是 Node Critical Pod (如静态 Pod)，设为 -997
    if types.IsNodeCriticalPod(pod) {
        return guaranteedOOMScoreAdj
    }

    switch v1qos.GetPodQOS(pod) {
    case v1.PodQOSGuaranteed:
        // 2. Guaranteed Pod 设为 -997
        return guaranteedOOMScoreAdj
    case v1.PodQOSBestEffort:
        // 3. BestEffort Pod 设为 1000
        return besteffortOOMScoreAdj
    }

    // 4. Burstable Pod 计算逻辑
    containerMemReq := container.Resources.Requests.Memory().Value()

    // 如果启用了 PodLevelResources 特性，计算方式会有所不同，这里展示默认逻辑
    // 公式：1000 - (1000 * containerMemReq / memoryCapacity)
    oomScoreAdjust := 1000 - (1000*containerMemReq)/memoryCapacity

    // 5. Sidecar 容器特殊处理 (省略部分细节)
    if isSidecarContainer(pod, container) {
        // ... 计算 sidecar 的 OOM score ...
    }

    // 6. 边界值修正
    // 保证 Burstable 的 OOM score 至少比 Guaranteed 高 (即 > 3)
    // (1000 + -997) = 3
    if int(oomScoreAdjust) < (1000 + guaranteedOOMScoreAdj) {
        return (1000 + guaranteedOOMScoreAdj)
    }
    // 保证 Burstable 的 OOM score 比 BestEffort 低 (即 < 1000)
    if int(oomScoreAdjust) == besteffortOOMScoreAdj {
        return int(oomScoreAdjust - 1)
    }
    return int(oomScoreAdjust)
}
```

---

## 5. QoS 与 节点压力驱逐 (Eviction)

除了内核触发的 OOM Kill，Kubelet 自身也会监控节点资源。当节点资源（内存、磁盘空间等）达到驱逐阈值（Eviction Threshold）时，Kubelet 会主动驱逐 Pod 以回收资源。

Kubelet 的驱逐策略同样依赖 QoS：

1. **BestEffort** Pod 是首选驱逐目标。
2. **Burstable** Pod 如果其实际使用量超过了 Request，则是次选目标。
3. **Guaranteed** Pod 和实际使用量低于 Request 的 **Burstable** Pod 是最后的目标。

---

## 6. 总结与最佳实践

Kubernetes 的 QoS 机制通过简单的资源配置（Request/Limit），在底层实现了复杂的资源隔离和保护策略。

| QoS 类别       | CPU 调度权重 | OOM 优先级  | 适用场景                                       |
| :------------- | :----------- | :---------- | :--------------------------------------------- |
| **Guaranteed** | 高 (按比例)  | 极低 (-997) | 核心数据库、状态敏感服务、需严格延迟保障的业务 |
| **Burstable**  | 中 (按比例)  | 中 (动态)   | Web 服务、微服务、有突发流量但需一定保障的业务 |
| **BestEffort** | 极低 (2/1)   | 极高 (1000) | 离线批处理、CI/CD 构建任务、开发测试环境       |

**最佳实践建议**：

- 对于生产环境的核心应用，尽量配置 **Guaranteed** QoS，即显式设置 Request 和 Limit 且相等，以防止被意外 Kill。
- 对于大多数无状态 Web 服务，**Burstable** 是最经济的选择。建议合理设置 Request 以保证基本运行，同时利用 Limit 限制最大影响范围。
- **BestEffort** 慎用于生产环境的关键路径，但非常适合用于填充集群的碎片资源，提高整体资源利用率。
