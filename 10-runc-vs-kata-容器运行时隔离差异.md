# runc vs Kata — 从两个真实 bug 理解容器运行时的隔离差异

> **背景**：JDOS kata-containers（`jdcloud` 分支）里有两类改动：
> - `3cdf9cfdc feat: 更新kata-agent的 oom_score_adj = -1000`
> - `44764ae5f / 4fa80d50a feature: kata容器增加RUNTIME/id/name等环境变量，提供给监控agent`
>
> 这两个 commit 只在 Kata 里才需要，runc 里不存在对应问题。
> 本文从这两个 bug 出发，彻底搞清楚 runc 和 Kata 的隔离机制差异。

---

## 一、理论基础

### 1. runc 是怎么工作的

runc 是 OCI 标准的参考实现，容器本质是**宿主机上的一组受限进程**：

```
宿主机内核
    │
    ├── namespace（隔离视图）
    │       ├── net ns → 容器独立 IP
    │       ├── pid ns → 容器 PID 从 1 开始
    │       ├── mnt ns → 容器独立 rootfs
    │       └── uts ns → 容器独立 hostname
    │
    └── cgroup（限制资源）
            ├── memory.limit_in_bytes = 512MB
            └── cpu.cfs_quota_us = 500m
```

**关键特征**：
- 容器进程和宿主机进程**共享同一个内核**
- 宿主机可以直接通过 `/proc/<pid>/` 看到容器内所有进程
- 容器的所有 syscall 直接打到宿主机内核
- 隔离靠 namespace，限制靠 cgroup，没有额外层

### 2. Kata 是怎么工作的

Kata Containers 给每个 Pod 起一个**轻量虚拟机（VM）**，容器跑在 VM 里：

```
宿主机内核
    │
    └── QEMU 进程（hypervisor）
            │ KVM 虚拟化
            └── 轻量 VM（Guest 内核）
                    │
                    ├── kata-agent（VM 内部的 daemon）
                    │       └── 负责在 VM 里创建容器（调 runc）
                    │
                    └── 容器进程（业务代码）
                            ├── namespace + cgroup（VM 内部的）
                            └── 只能发 syscall 给 Guest 内核
```

containerd 通过 **kata-shim**（shimv2 协议）和 QEMU 通信：

```
containerd
    ↓ shimv2（ttrpc over vsock）
kata-shim（宿主机上的进程）
    ↓ QMP（QEMU Monitor Protocol）
QEMU
    ↓ vsock
kata-agent（VM 内部）
    ↓ OCI runtime（mini runc）
容器进程
```

**关键特征**：
- 容器进程的 syscall 打给 **Guest 内核**，不直接接触宿主机内核
- 宿主机**看不到** VM 内部的进程（`/proc/<pid>/` 里没有）
- kata-agent 是 VM 内部唯一的管理进程，如果它死了，VM 里所有容器都挂
- 每个 Pod 一个独立 VM，隔离强度等同于虚拟机

### 3. 一张图对比核心差异

```
                runc                          Kata
               ───────                       ──────
进程可见性    宿主机 /proc 直接可见           宿主机看不到 VM 内部进程
内核共享      共享宿主机内核                  独立 Guest 内核
syscall 路径  进程 → 宿主机内核              进程 → Guest 内核 → KVM → 宿主机内核
隔离强度      namespace 边界（可突破内核漏洞） VM 边界（类虚拟机安全）
启动速度      < 100ms                        500ms ~ 1s（QEMU 启动）
内存开销      < 10MB overhead                ~100MB（Guest 内核 + kata-agent）
监控方式      宿主机直接 /proc 采集           必须在 VM 内部采集，或通过 vsock 透传
cgroup 位置   宿主机 cgroup 层级             宿主机侧只有 QEMU 进程的 cgroup
```

---

## 二、关键命令

### 判断当前容器是 runc 还是 Kata

```bash
# 方法 1：看 runtimeClassName
kubectl get pod <pod> -o jsonpath='{.spec.runtimeClassName}'
# 空 或 runc → runc
# kata / kata-qemu → Kata

# 方法 2：在容器里看内核版本
kubectl exec <pod> -- uname -r
# 和宿主机 uname -r 一样 → runc（共享内核）
# 不一样（通常更老更小）→ Kata（Guest 内核）

# 方法 3：看宿主机上有没有 QEMU 进程
ps aux | grep qemu | grep -v grep
# 有 → 有 Kata Pod 在跑

# 方法 4：看 containerd shim 类型
ps aux | grep containerd-shim
# containerd-shim-runc-v2 → runc
# containerd-shim-kata-v2 → Kata
```

### 查看 Kata VM 相关

```bash
# 找 kata Pod 对应的 QEMU 进程（看命令行参数里的 sandbox id）
ps aux | grep qemu

# 查看 kata-agent 是否在跑（在宿主机上看不到，需要进 VM）
# 通过 kata-runtime 工具连进 VM
kata-runtime exec <sandbox-id>

# 查看 QEMU 的 cgroup（宿主机侧只能看到 QEMU 进程）
cat /proc/$(pgrep qemu)/cgroup

# 在容器里查 cgroup（VM 内部的 cgroup，和宿主机的不同）
kubectl exec <kata-pod> -- cat /proc/self/cgroup
# 看到的是 Guest 内核里的 cgroup 路径，不是宿主机的
```

### runc 容器的进程可见性 vs Kata

```bash
# runc 容器：宿主机可以直接看到容器进程
kubectl exec <runc-pod> -- sleep 100 &
ps aux | grep sleep     # 宿主机上能看到这个 sleep 进程
ls /proc/$(pgrep sleep)/ns/net  # 能看到容器的 netns

# Kata 容器：宿主机看不到容器进程
kubectl exec <kata-pod> -- sleep 100 &
ps aux | grep sleep     # 宿主机上找不到！sleep 在 VM 里面
# 宿主机只能看到 QEMU 进程
```

### oom_score_adj 相关

```bash
# 查看进程的 OOM 分数（越高越容易被杀）
cat /proc/<pid>/oom_score
cat /proc/<pid>/oom_score_adj   # -1000 = 永不被杀，1000 = 最先被杀

# 查看 kata-agent 的 oom_score_adj（在 VM 内部）
kubectl exec <kata-pod> -- cat /proc/1/oom_score_adj
# 修复后应该是 -1000

# 宿主机上 QEMU 进程的 oom_score_adj
cat /proc/$(pgrep qemu)/oom_score_adj
```

---

## 三、工程实践 1：kata-agent oom_score_adj = -1000

### 背景

在内存紧张时，Linux 内核的 OOM killer 会选一个进程杀掉以释放内存。选择标准是 `oom_score`（越高越先死），计算公式大致是：进程自身内存占用越大、优先级越低 → oom_score 越高。

### 现象

节点内存压力大时，Kata Pod 突然全挂：

```
# 宿主机 dmesg 日志
Out of memory: Kill process 12345 (kata-agent) score 512 or sacrifice child
Killed process 12345 (kata-agent) total-vm:512000kB, anon-rss:128000kB
```

`kata-agent` 被 OOM killer 杀死了。

### 根因

**runc 里为什么不存在这个问题**：

在 runc 容器里，"容器的管理者"是宿主机上的 containerd，不在容器里面。OOM killer 杀的是容器内业务进程（`app`、`nginx` 等），不会影响 containerd 本身。

**Kata 里为什么会有问题**：

Kata 的容器管理者（kata-agent）跑在 **VM 内部**：

```
VM 内的进程：
  PID 1: kata-agent（内存占用：~50MB，OOM score 会被计算进去）
  PID 2: app（业务进程）

VM 内存总量有限（比如 512MB），
kata-agent 自己就占了 50MB，
OOM killer 计算 oom_score 时认为 kata-agent "很占内存"，
把它列为优先击杀目标。
```

一旦 kata-agent 死了：
- 它管的所有容器生命周期全部失控
- containerd 无法再通过 vsock 联系到 VM 内部
- 整个 Pod 的 containerd shim 挂掉

### Fix

```go
// kata-agent 启动时设置自己的 oom_score_adj
// src/agent/src/main.rs（kata-agent 是 Rust 写的）

fn protect_from_oom_killer() {
    let oom_score_adj_path = "/proc/self/oom_score_adj";
    fs::write(oom_score_adj_path, "-1000")
        .expect("failed to set oom_score_adj");
}
```

`-1000` 是 Linux 允许的最低值，代表"OOM killer 永远不杀我"。

同理，runc 容器里的 **`containerd`** 进程在宿主机上也应该设 oom_score_adj 为负值，道理一样——管理者不能比被管理的先死。

### 验证

```bash
# 进 Kata Pod 的容器看 kata-agent 的 oom_score_adj
# kata-agent 是 PID 1（VM 内部的 init 进程）
kubectl exec <kata-pod> -- cat /proc/1/oom_score_adj
# -1000   ← fix 后的值

# 对比 runc 容器：PID 1 是 pause 或用户进程，oom_score_adj 不一定是 -1000
kubectl exec <runc-pod> -- cat /proc/1/oom_score_adj
# 0（默认）或负数（如果用户设了 pod.spec.priorityClassName）
```

---

## 四、工程实践 2：kata容器注入 RUNTIME/id/name 环境变量

### 背景

JDOS 有一个监控 agent（类似 node-exporter 或 cAdvisor），需要知道每个容器的：
- 容器 ID（用于关联日志/指标）
- 容器名（展示给用户）
- 运行时类型（runc 还是 kata）

### 现象

监控 agent 采集 Kata Pod 的指标时，容器 ID/名字字段为空，或者无法区分 runc 和 kata：

```json
{
  "pod": "my-sandbox-xxx",
  "container": "",       // 空
  "runtime": "",         // 空
  "cpu_usage": 0.5
}
```

### 根因

**runc 容器里为什么容易采集**：

宿主机上的监控 agent 可以直接读 `/proc/<pid>/environ`（进程的环境变量）或者通过 containerd gRPC 查询容器 metadata：

```bash
# 宿主机上直接读 runc 容器的环境变量
cat /proc/<app-pid>/environ | tr '\0' '\n' | grep CONTAINER
# CONTAINER_NAME=app
# CONTAINER_ID=abc123
```

**Kata 容器里为什么难采集**：

Kata 容器的进程在 VM 里，宿主机的 `/proc` 里没有这些进程。

监控 agent 有两种采集策略：
1. **宿主机侧采集**：从 containerd API 查 → 能查到 Pod 级别，但查不到 VM 内部进程详情
2. **容器内部采集**：在容器里跑 sidecar 采集 → 但 sidecar 不知道自己叫什么名字

### Fix

在 Kata 容器启动时，containerd 向容器的环境变量里注入元数据：

```go
// JDOS containerd 的改动（伪代码）
// 创建 Kata 容器时，额外注入这些 env
extraEnvs := []string{
    fmt.Sprintf("RUNTIME=kata"),
    fmt.Sprintf("CONTAINER_ID=%s", containerID),
    fmt.Sprintf("CONTAINER_NAME=%s", containerName),
    fmt.Sprintf("POD_NAME=%s", podName),
    fmt.Sprintf("POD_NAMESPACE=%s", podNamespace),
}
config.Envs = append(config.Envs, extraEnvs...)
```

容器内的监控 agent 启动后读 `os.Getenv("RUNTIME")` 就知道自己在 Kata 里。

**为什么 runc 不需要这个**：

runc 容器里宿主机侧监控 agent 能直接查到一切，不需要依赖进程内部知道自己的 metadata。但 Kata 的 VM 壳阻断了这条路，只能靠"从外面注入、从里面读"来传递信息。

### 验证

```bash
# Kata 容器里看注入的环境变量
kubectl exec <kata-pod> -- env | grep -E "RUNTIME|CONTAINER_ID|CONTAINER_NAME"
# RUNTIME=kata
# CONTAINER_ID=abc123def456
# CONTAINER_NAME=app
```

---

## 五、两个 bug 的共同教训

| | runc | Kata |
|---|---|---|
| **管理者在哪** | 宿主机（containerd） | VM 内部（kata-agent）|
| **管理者能被 OOM 杀吗** | 不会（宿主机 containerd 优先级高）| 会（kata-agent 在 VM 内和业务争内存）|
| **宿主机能看到容器进程吗** | 能（/proc 直接可见） | 不能（在 VM 里）|
| **监控如何采集 metadata** | 宿主机直接读 /proc/environ | 必须提前注入到容器 env |
| **内核漏洞影响** | 容器逃逸可能打到宿主机内核 | 需先逃出 VM，隔离更强 |

**核心结论**：

Kata 的强隔离是"双刃剑"。VM 边界让容器更安全，但也让所有"宿主机直接看容器内部"的假设全部失效。凡是 JDOS kata 仓库里"在容器里注入 XXX / 在 kata-agent 里保护 YYY"这类 commit，背后都是同一个原因：**VM 壳把宿主机和容器隔开了，原来能走捷径的地方现在要绕路**。

---

## 六、延伸：cgroup 在 runc 和 Kata 里的位置差异

这是看 cgroup 相关 commit 时经常混淆的点：

```
runc 容器的 cgroup（宿主机侧）：
/sys/fs/cgroup/memory/kubepods/pod<pod-uid>/<container-id>/
    memory.limit_in_bytes = 536870912   ← 容器的内存限制
    memory.usage_in_bytes = 26529792    ← 容器的内存使用

Kata 容器的 cgroup（宿主机侧）：
/sys/fs/cgroup/memory/kubepods/pod<pod-uid>/<sandbox-id>/
    memory.limit_in_bytes = 536870912   ← QEMU 进程的内存限制
    memory.usage_in_bytes = 2147483648  ← QEMU + VM 内核 + 业务进程的总和

Kata 容器的 cgroup（VM 内侧，Guest cgroup）：
/sys/fs/cgroup/memory/kata-<container-id>/
    memory.limit_in_bytes = 536870912   ← 容器内部的限制（kata-agent 设的）
    memory.usage_in_bytes = 26529792    ← 业务进程真实使用
```

JDOS kata 仓库里 `de9f7ecf1 fix: 根据容器的注解决定memory.memsw.limit_in_bytes` 这类 commit，改的是 VM 内部（Guest 侧）的 cgroup 配置，不是宿主机侧的 QEMU cgroup。**搞清楚你在哪一侧操作**，是读这类 commit 的前提。

---

## 七、参考资料

- [Kata Containers 架构文档](https://github.com/kata-containers/kata-containers/blob/main/docs/design/architecture/README.md)
- [OCI Runtime Spec](https://github.com/opencontainers/runtime-spec/blob/main/spec.md)
- [Linux OOM killer 机制](https://www.kernel.org/doc/html/latest/mm/oom.html)
- [Linux /proc/[pid]/oom_score_adj](https://man7.org/linux/man-pages/man5/proc.5.html)
- JDOS kata commit：`3cdf9cfdc feat: 更新kata-agent的 oom_score_adj = -1000`
- JDOS kata commit：`44764ae5f feature: kata容器增加RUNTIME环境变量，提供给监控agent`
- JDOS kata commit：`4fa80d50a feature: kata容器增加id name等环境变量，提供给监控agent`
- Topic 7：[[07-cgroup和proc实战-容器看到宿主机内存的bug]]（cgroup 基础）
- Topic 8：[[08-namespace基础-pause容器持有netns与sysctl应用]]（namespace 基础）
