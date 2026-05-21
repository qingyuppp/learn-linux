# learn-linux

后端开发者必备的 Linux 基础技能，通过实操笔记学习。每个主题对应一个 markdown 文件，记录命令、用法和实际场景。

## 学习路径

### 基础篇

1. **文件与目录** — ls、cd、cp、mv、rm、find、chmod、chown
2. **文本处理** — grep、sed、awk、sort、uniq、wc、head、tail
3. **进程管理** — ps、top、kill、nohup、&、jobs
4. **网络排查** — curl、wget、netstat、ss、dig、nslookup、ping、traceroute
5. **系统信息** — df、du、free、uname、lsof
6. **Shell 脚本** — 变量、条件、循环、函数、实用脚本

### 容器原理篇（实战导向）

7. **[/proc 与 cgroup 实战](容器看到宿主机内存的bug-cgroup和proc实战.md)** — 从一个真实 bug 理解容器资源隔离
8. **namespace 基础** — PID、Network、Mount 等 6 种 namespace 的作用
9. **kubectl 排障套路** — exec、describe、logs、events 实战命令组合
10. **runc vs Kata 隔离差异** — 共享内核 vs 独立 VM 的可观测性区别

## 关联

本项目是 [后端开发技能路线图](https://github.com/qingyuppp/learn-api) 的第 4 项技能。

容器原理篇的素材来自 [OpenSandbox](https://github.com/alibaba/opensandbox) 实习工作中的真实排障经历。
