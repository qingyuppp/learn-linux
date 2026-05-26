# learn-linux

后端开发者必备的 Linux 基础技能，通过实操笔记学习。每个主题对应一个 markdown 文件，记录命令、用法和实际场景。

## 学习路径

### 基础篇

| # | 主题 | 涉及命令 | 状态 |
|---|------|---------|------|
| 1 | 文件与目录 | ls、cd、cp、mv、rm、find、chmod、chown | ⏳ 待写 |
| 2 | 文本处理 | grep、sed、awk、sort、uniq、wc、head、tail | ⏳ 待写 |
| 3 | 进程管理 | ps、top、kill、nohup、&、jobs | ⏳ 待写 |
| 4 | 网络排查 | curl、wget、netstat、ss、dig、nslookup、ping、traceroute | ⏳ 待写 |
| 5 | 系统信息 | df、du、free、uname、lsof | ⏳ 待写 |
| 6 | Shell 脚本 | 变量、条件、循环、函数、实用脚本 | ⏳ 待写 |

### 容器原理篇（实战导向）

| # | 主题 | 涉及概念/命令 | 状态 |
|---|------|--------------|------|
| 7 | [/proc 与 cgroup 实战](容器看到宿主机内存的bug-cgroup和proc实战.md) | /proc、cgroup v1/v2、namespace 隔离边界 | ✅ 完成 |
| 8 | [namespace 基础](namespace基础-pause容器持有netns与sysctl应用.md) | unshare、nsenter、lsns、PID/Network/Mount/UTS/IPC/User namespace | ✅ 完成 |
| 9 | kubectl 排障套路 | exec、describe、logs、events、port-forward、cp | ⏳ 待写 |
| 10 | runc vs Kata 隔离差异 | runc 共享内核、Kata 独立 VM、可观测性差异 | ⏳ 待写 |

### 进阶篇（深入内核与系统）

| # | 主题 | 涉及概念/命令 | 状态 |
|---|------|--------------|------|
| 11 | 系统调用追踪 | strace、ltrace、bpftrace | ⏳ 待写 |
| 12 | 性能分析 | perf、top、iostat、vmstat、sar、火焰图 | ⏳ 待写 |
| 13 | 文件描述符与 IO | lsof、文件锁、阻塞/非阻塞、epoll | ⏳ 待写 |
| 14 | 信号与进程通信 | signal、pipe、socket、shared memory | ⏳ 待写 |
| 15 | systemd 与服务管理 | systemctl、journalctl、unit 文件 | ⏳ 待写 |

### 网络篇

| # | 主题 | 涉及概念/命令 | 状态 |
|---|------|--------------|------|
| 16 | 网络命名空间实战 | ip netns、veth pair、bridge | ⏳ 待写 |
| 17 | iptables 与 nftables | 链/规则/NAT、Docker 网络原理 | ⏳ 待写 |
| 18 | TCP 排障 | tcpdump、wireshark、ss -i、抓包分析 | ⏳ 待写 |
| 19 | DNS 与服务发现 | resolv.conf、systemd-resolved、CoreDNS | ⏳ 待写 |

### 安全与权限篇

| # | 主题 | 涉及概念/命令 | 状态 |
|---|------|--------------|------|
| 20 | 用户与权限 | sudo、su、setuid、acl、umask | ⏳ 待写 |
| 21 | SELinux/AppArmor | 强制访问控制、容器安全策略 | ⏳ 待写 |
| 22 | capability 与 secure computing | capsh、seccomp、容器去 root | ⏳ 待写 |

## 关联

本项目是 [后端开发技能路线图](https://github.com/qingyuppp/learn-api) 的第 4 项技能。

容器原理篇的素材来自 [OpenSandbox](https://github.com/alibaba/opensandbox) 实习工作中的真实排障经历。

## 状态图例

- ✅ 完成
- 🔄 进行中
- ⏳ 待写

## 学习原则

1. **每篇都有真实工程案例**——不写"hello world"式的玩具示例
2. **理论 + 命令 + 工程实践**三段式结构
3. **来源可溯**——引用具体源码文件路径、行号、commit
