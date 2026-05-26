# namespace 基础 — 从一个 sysctl 应用错位 bug 理解 Linux 命名空间

> **背景**：JDOS containerd（`v1.7.5-jd` 分支）中有一个 fix：
> `17bb58732 fix: 仅在 Sandbox 容器中应用 net.* sysctl 参数`
> 为什么 sysctl 不能在 workload 容器里应用？
> 本文从这个 bug 出发，串联 Linux namespace 的核心机制。

---

## 一、理论基础

### 1. Linux namespace 是什么

namespace 是内核给进程提供的**资源视图隔离**机制——同一个内核上，不同 namespace 里的进程看到的"世界"不一样。

| namespace 类型 | 隔离的资源 | 典型用途 |
|---|---|---|
| **net**（Network） | 网络栈（IP/端口/路由/iptables） | 容器独立 IP |
| **pid**（PID） | 进程 ID 空间 | 容器内 PID 从 1 开始 |
| **mnt**（Mount） | 文件系统挂载点 | 容器独立 rootfs |
| **uts**（UTS） | hostname / domainname | 容器有自己的 hostname |
| **ipc**（IPC） | System V IPC、POSIX 消息队列 | 容器间 IPC 隔离 |
| **user**（User） | UID/GID 映射 | rootless 容器 |

**关键**：namespace 不是一个开关，而是一个**容器（inode）**。进程加入哪个 namespace，就看到那个 namespace 的资源视图。

### 2. namespace 的本质：一个内核对象

每个 namespace 在内核里是一个带引用计数的对象。只要有进程引用它（或者有 bind mount 指向它），它就存活。

```bash
# 查看当前进程所在的所有 namespace（每个都是一个 inode）
ls -la /proc/self/ns/

# 输出示例：
# lrwxrwxrwx 1 root root 0 May 25 net -> net:[4026531992]
# lrwxrwxrwx 1 root root 0 May 25 pid -> pid:[4026531836]
# lrwxrwxrwx 1 root root 0 May 25 mnt -> mnt:[4026531840]
# lrwxrwxrwx 1 root root 0 May 25 uts -> uts:[4026531838]

# 方括号里的数字是 inode 号。
# 两个进程如果 net inode 相同，说明它们共享同一个 network namespace。
```

**共享 namespace 的方式**：
- `clone()` 时不传 `CLONE_NEWNET`（子进程继承父进程的 ns）
- `setns(fd, CLONE_NEWNET)`（进程主动加入一个已有的 ns）
- `unshare(CLONE_NEWNET)`（进程脱离当前 ns，进入新的 ns）

### 3. pause 容器：Pod 的 namespace 锚点

这是理解 containerd fix 的关键。

K8s 的 Pod 里可以有多个容器，但它们共享网络栈（同一个 IP、同一套端口空间）。这是通过 **pause 容器**实现的：

```
Pod 启动时：

1. containerd 先起 pause 容器（也叫 sandbox 容器）
   └── pause 进程调 unshare(CLONE_NEWNET)
       └── 创建一个新的 network namespace（netns A）
       └── pause 进程持有这个 netns（只要 pause 活着，netns A 就活着）

2. CNI 插件收到通知，给 netns A 分配 IP（比如 10.0.0.5）
   └── 在 netns A 里配置 veth pair、路由表、iptables 规则

3. containerd 再起 workload 容器（业务容器）
   └── 调 setns(pause_netns_fd, CLONE_NEWNET)
       └── workload 容器加入 netns A（和 pause 共享）
       └── workload 容器不创建自己的 netns
```

结果：

```
Pod IP = 10.0.0.5
    ↑
    netns A（由 pause 创建并持有）
    ↑          ↑
  pause      workload-1   workload-2
（netns A 的 owner）（加入者）（加入者）
```

**pause 容器的唯一职责**：持有 namespace（net/uts/ipc），不干任何业务逻辑。

---

## 二、关键命令

### 查看 namespace

```bash
# 列出系统上所有 namespace（需要 root）
lsns

# 查看某个进程的所有 namespace inode
ls -la /proc/<pid>/ns/

# 查看两个进程是否在同一个 netns
ls -la /proc/<pid1>/ns/net /proc/<pid2>/ns/net
# 输出里的 inode 号相同 → 共享同一个 netns

# 查看容器的 pause 进程 PID（在宿主机上）
# 先找 Pod 里的 pause 容器 ID
crictl pods | grep <pod-name>
crictl inspectp <pod-id> | grep pid
```

### unshare：创建新 namespace

```bash
# 创建新的 network namespace，在里面跑 bash
sudo unshare --net bash
# 此时 bash 在一个全新的 netns 里，只有 lo 接口
ip a   # 只看到 lo

# 创建新的 PID + Mount namespace
sudo unshare --pid --mount-proc --fork bash
# 此时 bash 的 PID = 1，ps 只看到自己
ps aux
```

### nsenter：进入已有 namespace

```bash
# 进入 PID=<pid> 进程所在的 network namespace
sudo nsenter --net=/proc/<pid>/ns/net -- ip a

# 进入 pause 容器的 netns，查看容器的网络配置
# 1. 找 pause 进程的 PID
PAUSE_PID=$(crictl inspectp <pod-id> | python3 -c "import sys,json; print(json.load(sys.stdin)['info']['pid'])")
# 2. 进入 netns 看网络
sudo nsenter --net=/proc/$PAUSE_PID/ns/net -- ip a
sudo nsenter --net=/proc/$PAUSE_PID/ns/net -- ss -tlnp

# 进入 pause 的 netns，验证 sysctl 值
sudo nsenter --net=/proc/$PAUSE_PID/ns/net -- sysctl net.core.somaxconn
```

### ip netns：管理具名 netns

```bash
# 创建一个具名 netns（会在 /var/run/netns/ 创建 bind mount）
sudo ip netns add test-ns

# 在 test-ns 里跑命令
sudo ip netns exec test-ns ip a
sudo ip netns exec test-ns sysctl net.core.somaxconn

# 列出所有具名 netns
ip netns list

# 删除
sudo ip netns del test-ns
```

### sysctl 与 namespace

```bash
# net.* sysctl 是 network-namespace-scoped 的
# 在宿主机设置，只影响宿主机的 netns
sysctl net.core.somaxconn=128

# 在某个 netns 里设置，只影响那个 netns
sudo nsenter --net=/proc/<pause-pid>/ns/net -- sysctl -w net.core.somaxconn=1024
# 宿主机上的值不变，其他 Pod 的值不变

# 验证隔离性
sysctl net.core.somaxconn               # 宿主机
sudo nsenter --net=/proc/<pause-pid>/ns/net -- sysctl net.core.somaxconn  # Pod 里
# 两个值独立
```

---

## 三、工程实践：net.* sysctl 应用错位 bug

### 背景

K8s 允许用户通过 Pod annotation 给容器设置内核参数：

```yaml
apiVersion: v1
kind: Pod
metadata:
  annotations:
    security.alpha.kubernetes.io/unsafe-sysctls: "net.core.somaxconn=1024"
spec:
  containers:
  - name: app
    ...
```

containerd 在 CRI 层要把这个 annotation 里的 sysctl 值应用到容器的 netns 里。

### 现象

设置了 `net.core.somaxconn=1024`，但进到容器里验证：

```bash
kubectl exec <pod> -c app -- sysctl net.core.somaxconn
# net.core.somaxconn = 128    ← 还是默认值，没生效
```

### 排查

#### 第一步：理解 sysctl 的作用域

`net.*` 开头的 sysctl 都是 **network-namespace-scoped**：
- 每个 network namespace 有自己独立的一份 sysctl 值
- 在 netns A 里改，只影响 netns A，不影响 netns B

```bash
# 验证：宿主机和容器的 net.core.somaxconn 相互独立
sysctl net.core.somaxconn                    # 宿主机：128
nsenter --net=/proc/<pause-pid>/ns/net \
    sysctl net.core.somaxconn               # Pod：128（各自独立）
```

#### 第二步：找到 containerd 应用 sysctl 的时机

containerd 实现 CRI 的 `CreateContainer` 接口时，从 ContainerConfig 里读出 sysctl 列表，对每个 sysctl 写入对应容器的 netns。

Bug 所在：**对每个容器（包括 workload 容器）都执行 sysctl 写入**。

```
RunPodSandbox（创建 pause）→ 应用 sysctl 到 pause 的 netns ← ✅ 正确
CreateContainer（创建 app）→ 应用 sysctl 到 app 的 netns  ← ❌ 错误
```

**问题**：`app` 容器没有自己的 netns！

```bash
# 在宿主机上验证：pause 和 app 容器的 net inode 一样
ls -la /proc/<pause-pid>/ns/net
ls -la /proc/<app-pid>/ns/net
# net -> net:[4026532XXX]   ← 同一个 inode，共享同一个 netns
```

对 `app` 容器"应用 sysctl"实际上在对 **pause 的 netns**（已经设好的）再写一次，或者用了错误的 fd 路径，导致写入的目标不对甚至失败。

#### 第三步：找到正确的应用时机

`net.*` sysctl 必须在 **pause 容器的 netns 创建之后、workload 容器加入之前** 应用：

```
RunPodSandbox：
  1. 创建 pause 容器，建立 netns
  2. CNI 配置网络（分配 IP）
  3. ← 在这里应用 net.* sysctl（写入 pause 的 netns）← ✅
  4. 返回 sandbox_id

CreateContainer（app）：
  1. 调 setns 加入 pause 的 netns（此时 sysctl 已经生效）
  2. 启动 app 进程
  ← 不要在这里再应用 sysctl ← fix 的核心
```

### Fix

```go
// 伪代码：fix 的逻辑
func (c *criService) CreateContainer(ctx, req) {
    containerConfig := req.GetConfig()
    sysctls := containerConfig.GetLinux().GetSysctls()

    for key, val := range sysctls {
        if strings.HasPrefix(key, "net.") {
            // ❌ 修复前：对每个容器都应用 net.* sysctl
            // ✅ 修复后：net.* 只在 sandbox 容器（pause）创建时应用，
            //    workload 容器跳过
            if !isSandboxContainer(containerConfig) {
                continue  // workload 容器跳过 net.* sysctl
            }
        }
        applySysctl(netnsFd, key, val)
    }
}
```

### 验证 fix 是否生效

```bash
# 1. 部署带 sysctl annotation 的 Pod
kubectl apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: sysctl-test
  annotations:
    security.alpha.kubernetes.io/unsafe-sysctls: "net.core.somaxconn=4096"
spec:
  containers:
  - name: app
    image: busybox
    command: ["sleep", "3600"]
EOF

# 2. 进 pause 的 netns 验证
PAUSE_PID=$(crictl inspectp $(crictl pods | grep sysctl-test | awk '{print $1}') \
    | python3 -c "import sys,json; print(json.load(sys.stdin)['info']['pid'])")
sudo nsenter --net=/proc/$PAUSE_PID/ns/net -- sysctl net.core.somaxconn
# net.core.somaxconn = 4096   ← ✅ 生效

# 3. 进 app 容器验证（共享同一个 netns，应该看到同样的值）
kubectl exec sysctl-test -c app -- sysctl net.core.somaxconn
# net.core.somaxconn = 4096   ← ✅ 生效

# 4. 验证宿主机不受影响
sysctl net.core.somaxconn
# net.core.somaxconn = 128    ← ✅ 宿主机没变
```

---

## 四、延伸：pause 被杀死会怎样

```
原地重启（reuse sandbox）的背景：

正常情况（pause 被杀）：
  pause 进程退出
    → netns 引用计数归零
    → netns 被内核回收
    → IP 消失、所有 workload 容器的网络连接断开
    → kubelet 重建 pause（新 netns，CNI 重新分配 IP）
    → IP 可能变化

原地重启（pause reuse，团队实现的功能）：
  pause 进程退出
    → containerd 不回收 netns（通过 bind mount 保住引用）
    → 重建 pause，setns 加入同一个 netns
    → IP 不变、网络连接不断
    → workload 容器重启后网络无感
```

这就是团队大量 commit 围绕"原地重启"的底层机制：
- `39a084627 fix: 当原地重启sandbox时IP可能会变需要更新存储`
- `7d6901529 根目录保持功能：关机重启后复用Pause容器，保持IP、netns、根目录`

理解了 pause 是 netns 的 holder、netns 靠引用计数存活，这些 commit 就能看懂了。

---

## 五、教训

1. **workload 容器没有自己的 netns**——它们 `setns` 进入 pause 的 netns，是加入者不是 owner
2. **net.* sysctl 是 netns-scoped 的**——要在 pause 的 netns 里设，不能在 workload 容器里设
3. **namespace 的生命周期靠引用计数**——pause 进程是 netns 的最后一个持有者，pause 死了 netns 就消失
4. **`nsenter` 是验证 namespace 归属的最直接工具**——有疑问就 `nsenter --net=/proc/<pid>/ns/net -- <cmd>` 进去看
5. **两个进程 `/proc/<pid>/ns/net` 的 inode 相同 = 共享同一个 netns**——这是快速验证的方法

---

## 六、参考资料

- [Linux man page: namespaces(7)](https://man7.org/linux/man-pages/man7/namespaces.7.html)
- [Linux man page: network_namespaces(7)](https://man7.org/linux/man-pages/man7/network_namespaces.7.html)
- [Linux man page: nsenter(1)](https://man7.org/linux/man-pages/man1/nsenter.1.html)
- [Linux man page: unshare(1)](https://man7.org/linux/man-pages/man1/unshare.1.html)
- [sysctl namespacing in Linux](https://www.kernel.org/doc/html/latest/admin-guide/sysctl/net.html)
- JDOS containerd commit：`17bb58732 fix: 仅在 Sandbox 容器中应用 net.* sysctl 参数`
- JDOS containerd commit：`39a084627 fix: 当原地重启sandbox时IP可能会变需要更新存储`
- [Kubernetes Pause Container](https://www.ianlewis.org/en/almighty-pause-container) — pause 容器的权威解读
