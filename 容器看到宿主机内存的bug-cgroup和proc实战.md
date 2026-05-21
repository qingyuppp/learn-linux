# /proc 与 cgroup 实战 — 从一个真实 bug 理解容器资源隔离

> **背景**：实习项目 OpenSandbox 中，发现 execd 服务的 `/metrics` 接口在 runc 容器里返回了宿主机的内存数据（28GB）而不是容器的内存配额（512MB）。本文从这个 bug 出发，串联 /proc、cgroup、namespace 三个 Linux 核心概念。

## 一、理论基础

### 1. /proc 文件系统是什么

`/proc` 是 Linux 内核暴露的**虚拟文件系统**，不占磁盘，里面的"文件"其实是内核数据结构的实时映射。

| 路径 | 内容 |
|------|------|
| `/proc/meminfo` | 系统内存总量、可用、缓存 |
| `/proc/stat` | CPU 时间统计、上下文切换 |
| `/proc/cpuinfo` | CPU 型号、核数 |
| `/proc/[pid]/` | 某个进程的详细信息 |
| `/proc/net/dev` | 网络接口的流量统计 |
| `/proc/net/tcp` | TCP 连接列表 |

`/proc` 是**系统监控工具（top、free、ps）的数据源**，gopsutil、prometheus node_exporter 这些库底层都在读 /proc。

### 2. cgroup 是什么

cgroup（control group）是 Linux 内核的**资源控制和统计机制**。容器运行时（runc、Kata）用 cgroup 来：

1. **限制**：CPU 配额、内存上限、IO 带宽
2. **统计**：实际用了多少 CPU/内存
3. **隔离**：不同 cgroup 互不影响

cgroup 有两个版本：

| 版本 | 路径 | 特点 |
|------|------|------|
| **v1** | `/sys/fs/cgroup/<controller>/...` | 每个资源（memory、cpu、net）一个独立目录 |
| **v2** | `/sys/fs/cgroup/...` | 统一层级，所有资源在一个目录树里 |

查看当前系统用的哪个版本：

```bash
# 如果输出 cgroup2，是 v2
stat -fc %T /sys/fs/cgroup/

# 或者看挂载点
mount | grep cgroup
```

### 3. namespace 与 /proc 的关系

namespace 是 Linux 的另一个隔离机制。容器创建时会进入新的 namespace，让进程"看到"独立的资源视图。

关键问题：**哪些 /proc 数据是 namespace 隔离的，哪些不是？**

| 数据 | 是否隔离 | 原因 |
|------|---------|------|
| `/proc/[pid]/` | ✓ PID namespace 隔离 | 只看得到本 namespace 的进程 |
| `/proc/net/dev` | ✓ network namespace 隔离 | 容器有独立网络栈 |
| `/proc/net/tcp` | ✓ network namespace 隔离 | 容器的 TCP 连接表 |
| **`/proc/meminfo`** | ❌ **不隔离** | **共享宿主机内存信息** |
| **`/proc/stat`（CPU）** | ❌ **不隔离** | **共享宿主机 CPU 信息** |
| `/proc/cpuinfo` | ❌ 不隔离 | 显示宿主机 CPU |

**这就是为什么容器内 `free -h` 看到的是宿主机内存**——`/proc/meminfo` 没有 namespace 隔离机制。

---

## 二、关键命令

### /proc 相关

```bash
# 查看系统内存（容器内看到的是宿主机）
cat /proc/meminfo | head -5

# 查看 CPU 统计
cat /proc/stat | head -1

# 查看当前进程的所有信息
cat /proc/self/status

# 查看某个进程占用的内存
cat /proc/$(pgrep nginx)/status | grep VmRSS
```

### cgroup v1 相关

```bash
# 内存使用量（字节）
cat /sys/fs/cgroup/memory/memory.usage_in_bytes

# 内存上限
cat /sys/fs/cgroup/memory/memory.limit_in_bytes

# CPU 累计使用（纳秒）
cat /sys/fs/cgroup/cpuacct/cpuacct.usage

# CPU 配额（quota / period = 核数）
cat /sys/fs/cgroup/cpu/cpu.cfs_quota_us   # 50000 表示 50ms
cat /sys/fs/cgroup/cpu/cpu.cfs_period_us  # 100000 表示 100ms
# 上面两个相除：50000/100000 = 0.5 核
```

### cgroup v2 相关

```bash
# 内存
cat /sys/fs/cgroup/memory.current  # 已用
cat /sys/fs/cgroup/memory.max      # 上限

# CPU
cat /sys/fs/cgroup/cpu.stat        # 包含 usage_usec
cat /sys/fs/cgroup/cpu.max         # 格式：quota period
```

### namespace 相关

```bash
# 查看进程的 namespace
ls -l /proc/self/ns/

# 输出示例（每个 ns 是一个 inode 链接）：
# mnt -> mnt:[4026531840]
# net -> net:[4026531992]
# pid -> pid:[4026531836]
# user -> user:[4026531837]

# 创建一个新的 PID + Mount namespace 玩玩
unshare --pid --mount-proc --fork bash
# 进去后 ps 只看到 bash 和 ps 两个进程
```

### 容器实战命令

```bash
# 进入容器看 /proc
kubectl exec -n <ns> <pod> -c <container> -- cat /proc/meminfo

# 在容器内读 cgroup
kubectl exec -n <ns> <pod> -c <container> -- \
  cat /sys/fs/cgroup/memory/memory.usage_in_bytes
```

---

## 三、工程实践：OpenSandbox /metrics bug

### 现象

OpenSandbox 的 execd 服务暴露了一个 `/metrics` JSON 接口给 SDK 用户查询沙箱资源：

```bash
$ kubectl exec sandbox-pod -c sandbox -- curl localhost:44772/metrics
{
  "cpu_count": 176,           # 宿主机 176 核（容器配额其实只有 0.5 核）
  "cpu_used_pct": 0.09,       # 宿主机 CPU 使用率（不是容器的）
  "mem_total_mib": 28884,     # 宿主机 28GB 内存（容器配额其实只有 512MB）
  "mem_used_mib": 28868       # 宿主机已用内存
}
```

**容器配额是 500m CPU / 512Mi 内存，但接口返回的是宿主机数据**。

### 排查过程

#### 第一步：定位代码

execd 源码在 `components/execd/pkg/web/controller/metric.go`：

```go
func (c *MetricController) readMetrics() (*model.Metrics, error) {
    metric := model.NewMetrics()
    metric.CpuCount = float64(runtime.GOMAXPROCS(-1))           // 176
    cpuPercent, _ := cpu.Percent(time.Second, false)            // gopsutil 读 /proc/stat
    metric.CpuUsedPct = cpuPercent[0]
    vmStat, _ := mem.VirtualMemory()                            // gopsutil 读 /proc/meminfo
    metric.MemTotalMiB = float64(vmStat.Total) / 1024 / 1024
    metric.MemUsedMiB = float64(vmStat.Used) / 1024 / 1024
    return metric, nil
}
```

代码用了 [gopsutil](https://github.com/shirou/gopsutil) 库的 `mem.VirtualMemory()`。

#### 第二步：追踪 gopsutil 实现

```
mem.VirtualMemory()
  → VirtualMemoryWithContext()
  → fillFromMeminfoWithContext()
  → common.HostProc("meminfo")  // 即 "/proc/meminfo"
```

gopsutil 读的是 `/proc/meminfo`。

#### 第三步：理解为什么 /proc/meminfo 是宿主机数据

在容器里手动验证：

```bash
# 在宿主机
$ cat /proc/meminfo | head -3
MemTotal:       29569024 kB
MemFree:           54392 kB
MemAvailable:     752432 kB

# 在 runc 容器里
$ kubectl exec sandbox-pod -- cat /proc/meminfo | head -3
MemTotal:       29569024 kB    # 一模一样！
MemFree:           54392 kB
MemAvailable:     752432 kB
```

**runc 容器共享宿主机的 /proc**——这就是根因。/proc/meminfo 不是 namespace 隔离的，容器内读到的就是宿主机数据。

#### 第四步：找到正确数据源

容器自己的内存数据在 cgroup 里：

```bash
$ kubectl exec sandbox-pod -- cat /sys/fs/cgroup/memory/memory.usage_in_bytes
26529792   # 25MB，容器真实使用

$ kubectl exec sandbox-pod -- cat /sys/fs/cgroup/memory/memory.limit_in_bytes
536870912  # 512MB，容器配额
```

### 修复方案

把 `mem.VirtualMemory()` 换成读 cgroup，兼容 v1/v2，fallback 到 gopsutil：

```go
func readCgroupMemory() (used, limit uint64, err error) {
    // cgroup v2
    if data, err := os.ReadFile("/sys/fs/cgroup/memory.current"); err == nil {
        used, _ = strconv.ParseUint(strings.TrimSpace(string(data)), 10, 64)
        if data, err := os.ReadFile("/sys/fs/cgroup/memory.max"); err == nil {
            limit, _ = strconv.ParseUint(strings.TrimSpace(string(data)), 10, 64)
        }
        return used, limit, nil
    }
    // cgroup v1
    if data, err := os.ReadFile("/sys/fs/cgroup/memory/memory.usage_in_bytes"); err == nil {
        used, _ = strconv.ParseUint(strings.TrimSpace(string(data)), 10, 64)
        if data, err := os.ReadFile("/sys/fs/cgroup/memory/memory.limit_in_bytes"); err == nil {
            limit, _ = strconv.ParseUint(strings.TrimSpace(string(data)), 10, 64)
        }
        return used, limit, nil
    }
    // fallback：非容器环境用 gopsutil
    vm, err := mem.VirtualMemory()
    if err != nil {
        return 0, 0, err
    }
    return vm.Used, vm.Total, nil
}
```

修复后接口返回：

```json
{
  "cpu_count": 0.5,
  "cpu_used_pct": 2.1,
  "mem_total_mib": 512,    // ✓ 容器配额
  "mem_used_mib": 25       // ✓ 容器真实使用
}
```

### Kata 模式的对比

切到 Kata 运行时（每个沙箱是独立 VM）：

```
runc 容器: /proc → 宿主机的 /proc       gopsutil 错 ❌
Kata VM:   /proc → VM 自己的 /proc       gopsutil 对 ✓
```

cgroup 方案对两种运行时都正确，是更通用的方案。

---

## 教训

1. **容器内不要盲信 /proc 的系统级数据**（meminfo、stat、cpuinfo）——这些不受 namespace 隔离
2. **进程级 /proc 数据是安全的**（/proc/self/、/proc/[pid]/）——PID namespace 隔离了
3. **网络相关 /proc 数据是安全的**（/proc/net/）——network namespace 隔离了
4. **资源限制和统计的权威数据源是 cgroup**，不是 /proc
5. **写 Go 库要注意运行环境**：gopsutil 在 VM/裸机/Kata 里对，在 runc 容器里错

## 延伸阅读

- [Linux Programmer's Manual: proc(5)](https://man7.org/linux/man-pages/man5/proc.5.html)
- [Control Group v2 文档](https://www.kernel.org/doc/Documentation/admin-guide/cgroup-v2.rst)
- [gopsutil 源码](https://github.com/shirou/gopsutil) — 看 `mem/mem_linux.go`
- [OpenSandbox execd metric.go](https://github.com/alibaba/opensandbox/blob/main/components/execd/pkg/web/controller/metric.go)
