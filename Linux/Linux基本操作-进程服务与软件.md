---
title: "Linux 基本操作（二）：进程、服务与软件"
date: "2026-06-15T00:00:00+08:00"
tags:
  - Linux
  - RHEL
  - 进程
  - 软件管理
description: "接续 Linux 基本操作系列，介绍进程查看与控制、systemd 服务管理和 RHEL 软件包管理，保留常用命令与操作示例，便于日常维护时查阅。"
---

上一篇：[Linux 基本操作（一）：文件、用户与权限](/articles/Linux/Linux基本操作/)。

# 五、进程与服务管理

## 1.进程和线程

程序是静态的代码文件，进程是程序运行后的实例。进程有独立的地址空间、文件描述符和资源。线程是进程内部的执行单元，同一进程内的线程共享进程资源，但每个线程有自己的栈和执行状态。

进程状态常见标记：

| 状态 | 含义 |
| --- | --- |
| `R` | 运行或可运行 |
| `S` | 可中断睡眠 |
| `D` | 不可中断睡眠，常见于 I/O 等待 |
| `T` | 暂停或被跟踪 |
| `Z` | 僵尸进程 |

## 2.查看进程

```bash
ps aux
ps -ef
ps axjf
ps -eo pid,ppid,user,stat,ni,pri,pcpu,pmem,comm --sort=-pcpu

pgrep sshd
pgrep -a sshd
pidof sshd
top
```

`ps aux` 常见字段：

| 字段 | 含义 |
| --- | --- |
| USER | 进程所有者 |
| PID | 进程 ID |
| %CPU | CPU 占用 |
| %MEM | 内存占用 |
| VSZ | 虚拟内存大小 |
| RSS | 常驻内存大小 |
| TTY | 关联终端 |
| STAT | 进程状态 |
| START | 启动时间 |
| TIME | 累计 CPU 时间 |
| COMMAND | 命令 |

`top` 内部常用操作：

| 按键 | 作用 |
| --- | --- |
| `P` | 按 CPU 排序 |
| `M` | 按内存排序 |
| `T` | 按累计 CPU 时间排序 |
| `1` | 展示每个 CPU 核心 |
| `k` | 向进程发送信号 |
| `u` | 只看指定用户进程 |
| `q` | 退出 |

## 3.前后台任务

```bash
command &
jobs
Ctrl+Z
bg %1
fg %1
```

`Ctrl+Z` 是暂停当前前台任务，`bg` 让暂停任务在后台继续运行，`fg` 把后台任务调回前台。

## 4.进程优先级

Linux 中普通用户可调的 nice 值范围是 `-20` 到 `19`，数值越小优先级越高。

```bash
nice -n 5 command
renice -n 10 -p PID
```

优先级标记：

| 标记 | 含义 |
| --- | --- |
| `<` | 高优先级 |
| `N` | 低优先级 |
| `+` | 前台进程组 |

## 5.进程信号

信号是 Linux 进程之间传递异步通知的机制。

| 信号 | 编号 | 默认行为 | 常见用途 |
| --- | --- | --- | --- |
| SIGHUP | 1 | 终止 | 终端断开、服务重新加载配置 |
| SIGINT | 2 | 终止 | `Ctrl+C` |
| SIGQUIT | 3 | 终止并转储 | `Ctrl+\` |
| SIGKILL | 9 | 强制终止 | 无法被捕获、阻塞或忽略 |
| SIGTERM | 15 | 终止 | 默认优雅结束进程 |
| SIGCONT | 18 | 继续 | 恢复暂停进程 |
| SIGSTOP | 19 | 暂停 | 无法被捕获、阻塞或忽略 |
| SIGTSTP | 20 | 暂停 | `Ctrl+Z` |

```bash
kill -15 PID
kill -9 PID
killall -15 process_name
pkill -u username
pkill -f "command pattern"
```

优先使用 `SIGTERM`，只有进程无法正常退出时再使用 `SIGKILL`。

## 6.systemd 服务管理

守护进程是在后台长期运行的服务进程，RHEL 9 使用 systemd 管理系统启动和服务。

```bash
systemctl status sshd
systemctl start sshd
systemctl stop sshd
systemctl restart sshd
systemctl reload sshd
systemctl enable sshd
systemctl enable --now sshd
systemctl disable sshd
systemctl list-units --type=service
systemctl list-unit-files
systemctl list-dependencies sshd
```

冻结与解冻服务：

```bash
systemctl mask service
systemctl unmask service
```

系统运行目标：

```bash
systemctl get-default
systemctl set-default multi-user.target
systemctl set-default graphical.target
systemctl isolate multi-user.target
systemctl isolate graphical.target
```

传统运行级别与 systemd target 的关系：

| 传统级别 | systemd target | 含义 |
| --- | --- | --- |
| 0 | `poweroff.target` | 关机 |
| 1 | `rescue.target` | 救援模式 |
| 3 | `multi-user.target` | 多用户无图形模式 |
| 5 | `graphical.target` | 图形模式 |
| 6 | `reboot.target` | 重启 |

查看服务日志：

```bash
journalctl -u sshd
journalctl -u sshd -f
journalctl -xe
```

# 六、软件管理

## 1.RHEL 9 软件管理体系

RHEL 9 常用 `rpm` 和 `dnf` 管理软件：

| 工具 | 作用 |
| --- | --- |
| `rpm` | 管理本地 rpm 包，适合查询、安装单个包 |
| `dnf` | 自动处理仓库、依赖、安装、更新和卸载 |

RHEL 9 常见仓库：

| 仓库 | 作用 |
| --- | --- |
| BaseOS | 操作系统核心组件 |
| AppStream | 应用程序、语言运行时、数据库等 |
| CRB | CodeReady Builder，开发库和构建依赖 |
| EPEL | Fedora 社区为 RHEL 生态提供的额外软件源 |

未注册的 RHEL 系统无法直接使用 Red Hat 官方订阅仓库。实验环境可使用 Red Hat Developer Subscription、离线 ISO 仓库，或使用兼容发行版仓库进行练习。

## 2.订阅提示与插件

原文提到为了避免未订阅系统的订阅管理提示，可以禁用订阅管理器插件。更准确的做法是修改插件配置，而不是把错误拼写写入 `dnf.cong`。

只禁用 subscription-manager 插件：

```bash
vim /etc/dnf/plugins/subscription-manager.conf
```

将配置改为：

```ini
[main]
enabled=0
```

不建议直接关闭所有 DNF 插件；若确实需要全局关闭插件，应修改 `/etc/dnf/dnf.conf` 中的 `plugins=0`，但这会影响其他插件。

## 3.rpm 常用命令

```bash
rpm -ivh package.rpm        # 安装
rpm -Uvh package.rpm        # 升级或安装
rpm -e package              # 卸载
rpm -qa                     # 查看所有已安装包
rpm -q package              # 查询包是否安装
rpm -ql package             # 查看包安装了哪些文件
rpm -qc package             # 查看包配置文件
rpm -qf /path/file          # 查询文件来自哪个包
rpm -qi package             # 查看包信息
```

`rpm` 不擅长自动解决依赖，日常优先使用 `dnf`。

## 4.dnf 常用命令

```bash
dnf repolist
dnf search nginx
dnf info nginx
dnf install nginx -y
dnf remove nginx -y
dnf update -y
dnf list installed
dnf provides /usr/sbin/semanage
dnf history
dnf history undo ID
dnf clean all
dnf makecache
```

软件包组：

```bash
dnf group list
dnf group info "Server with GUI"
dnf group install "Development Tools" -y
```

## 5.配置本地 ISO 仓库

实验环境中，如果 RHEL 未订阅，可以把安装 ISO 挂载为本地仓库：

```bash
mkdir -p /mnt/rhel9
mount /dev/sr0 /mnt/rhel9
```

创建仓库文件：

```bash
vim /etc/yum.repos.d/rhel9-local.repo
```

内容示例：

```ini
[BaseOS]
name=RHEL9 BaseOS
baseurl=file:///mnt/rhel9/BaseOS
enabled=1
gpgcheck=0

[AppStream]
name=RHEL9 AppStream
baseurl=file:///mnt/rhel9/AppStream
enabled=1
gpgcheck=0
```

刷新缓存：

```bash
dnf clean all
dnf makecache
dnf repolist
```

下一篇：[Linux 基本操作（三）：网络、传输与排障](/articles/Linux/Linux基本操作-网络传输与排障/)。
