---
title: Linux基础
date: 2026-06-15 00:00:00
tags:
  - Linux
  - RHEL
---

本文基于 **Red Hat Enterprise Linux 9.6** 整理。Red Hat 官方发布信息显示，RHEL 9.6 的 GA 日期为 **2025-05-20**，随版本发布的内核为 **5.14.0-570.12.1.el9_6**。需要注意，系统安装后若执行过更新，实际运行内核可能高于这个版本，应以本机命令结果为准：

```bash
cat /etc/redhat-release
uname -r
```

<!-- more -->

# 一、学习 Linux 时先建立的框架

Linux 的基础学习可以按一条主线理解：系统如何识别用户，用户如何访问文件，程序如何变成进程，服务如何长期运行，系统如何联网、安装软件和传输数据。

| 模块 | 要解决的问题 | 常用命令 |
| --- | --- | --- |
| 帮助手册 | 不记死命令，知道如何查 | `man`、`whatis`、`--help` |
| 文件系统 | 文件在哪里、如何管理 | `ls`、`cp`、`mv`、`rm`、`find`、`ln` |
| 用户管理 | 谁在操作系统 | `useradd`、`usermod`、`passwd`、`su`、`sudo` |
| 权限管理 | 谁能读、写、执行什么 | `chmod`、`chown`、`setfacl`、`chattr` |
| 进程管理 | 程序如何运行和停止 | `ps`、`top`、`pgrep`、`kill` |
| 服务管理 | 后台服务如何开关和自启 | `systemctl`、`journalctl` |
| 软件管理 | 软件从哪里安装和更新 | `rpm`、`dnf` |
| 网络管理 | 主机如何接入网络 | `ip`、`nmcli`、`ping`、`ss` |
| 文件传输 | 主机之间如何交换文件 | `scp`、`rsync`、`tar` |

在实验时，建议优先参考系统自带手册：

```bash
mandb
whatis command
command --help
man command
```

# 二、Linux 文件系统与常用操作

## 1.目录结构

Linux 采用单根目录结构，所有路径都从 `/` 开始。

| 目录 | 作用 |
| --- | --- |
| `/` | 根目录，所有目录的起点 |
| `/boot` | 内核、引导加载器相关文件 |
| `/etc` | 系统和服务配置文件 |
| `/home` | 普通用户家目录 |
| `/root` | root 用户家目录 |
| `/usr` | 系统程序、库、文档等主要软件资源 |
| `/var` | 经常变化的数据，如日志、缓存、邮件 |
| `/tmp` | 临时文件目录 |
| `/dev` | 设备文件 |
| `/proc` | 内核和进程的虚拟文件系统 |
| `/run` | 系统启动后生成的运行时数据 |
| `/mnt`、`/media` | 临时挂载点 |

路径分为绝对路径和相对路径：

```bash
/etc/passwd      # 绝对路径，从 / 开始
../file          # 相对路径，相对当前目录
.                # 当前目录
..               # 上级目录
~                # 当前用户家目录
```

## 2.常用文件命令

```bash
pwd                         # 查看当前目录
ls -lah                     # 查看文件列表和隐藏文件
cd /etc                     # 切换目录
mkdir -p dir/subdir          # 创建多级目录
touch file                  # 创建空文件或更新时间戳
cp file newfile             # 复制文件
cp -a dir dir.bak            # 归档方式复制目录，尽量保留属性
mv old new                  # 移动或重命名
rm -i file                  # 删除文件，交互确认
rm -r dir                   # 删除目录
file filename               # 判断文件类型
stat filename               # 查看文件元数据
du -sh dir                  # 查看目录占用空间
df -h                       # 查看文件系统容量
```

查找文件常用 `find`：

```bash
find /etc -name "*.conf"
find /var/log -type f -size +100M
find /home -user lehuang
```

## 3.链接文件

Linux 中常见硬链接和软链接：

```bash
ln source hardlink
ln -s source softlink
```

| 类型 | 特点 |
| --- | --- |
| 硬链接 | 指向同一个 inode，不能跨文件系统，通常不能给目录创建 |
| 软链接 | 类似快捷方式，保存目标路径，可跨文件系统，可指向目录 |

# 三、用户与用户组管理

## 1.用户、用户组和 3A 机制

Linux 通过用户和用户组管理系统资源。用户解决“谁在操作”，用户组解决“如何批量授权”。

Linux 安全管理可用 3A 机制理解：

| 机制 | 含义 |
| --- | --- |
| Authentication | 身份认证，确认用户是谁 |
| Authorization | 授权，决定用户能做什么 |
| Accounting | 审计，记录用户做了什么 |

用户组有两类：

| 类型 | 说明 |
| --- | --- |
| 主组 | 用户创建后默认所属的组 |
| 附加组 | 为了获得额外权限而加入的其他组 |

## 2.UID 和 GID

Linux 内部通过 UID 和 GID 识别用户和用户组。

| UID 区间 | 类型 | 说明 |
| --- | --- | --- |
| 0 | 超级用户 | `root`，拥有系统最高权限 |
| 1-999 | 系统用户 | 用于运行系统服务，通常不可交互登录 |
| 1000-65535 | 普通用户 | 日常登录和操作使用 |

不同发行版的区间可能略有差异，可以查看配置：

```bash
grep -E 'UID_MIN|UID_MAX|SYS_UID_MIN|SYS_UID_MAX' /etc/login.defs
grep -E 'GID_MIN|GID_MAX|SYS_GID_MIN|SYS_GID_MAX' /etc/login.defs
```

## 3.用户相关配置文件

| 文件或目录 | 作用 |
| --- | --- |
| `/etc/passwd` | 用户基本信息，所有用户可读 |
| `/etc/group` | 用户组基本信息 |
| `/etc/shadow` | 用户密码哈希和密码有效期，仅 root 可读 |
| `/etc/skel/` | 新用户家目录模板 |
| `/home/username` | 普通用户家目录 |
| `/var/spool/mail/username` | 用户本地邮件文件 |

`/etc/passwd` 一行有 7 列：

```text
用户名:密码占位符:UID:GID:注释:家目录:默认Shell
```

`/etc/group` 一行常见 4 列：

```text
组名:密码占位符:GID:附加成员列表
```

`/etc/shadow` 保存密码哈希和账号有效期，不应随意手工编辑。需要管理密码时优先使用 `passwd` 或 `chage`。

## 4.查看与切换用户

```bash
whoami
id
id lehuang
id -u lehuang
id -g lehuang
id -G lehuang

su username       # 切换身份，不完整切换环境
su - username     # 切换身份并加载目标用户环境
exit              # 退出当前用户身份
```

做用户切换时建议使用 `su - username`，使用完及时 `exit`，不要在同一个 Shell 中反复嵌套切换。

## 5.创建、修改和删除用户

```bash
useradd username
useradd -u 2000 -g groupname -G wheel -d /home/username -s /bin/bash username

userdel username
userdel -r username

groupadd groupname
groupadd -g 3000 groupname
groupdel groupname
```

常用 `usermod` 参数：

```bash
usermod -l newname oldname       # 修改用户名
usermod -u 2001 username         # 修改 UID
usermod -g group username        # 修改主组
usermod -G group1,group2 user    # 重设附加组
usermod -aG wheel username       # 追加附加组
usermod -d /new/home -m user     # 修改并迁移家目录
usermod -s /bin/bash username    # 修改默认 Shell
usermod -L username              # 锁定用户
usermod -U username              # 解锁用户
```

观察用户和组变化：

```bash
watch -n 1 "tail -n 3 /etc/passwd /etc/group; ls -l /home"
```

## 6.密码和账号有效期

```bash
passwd username          # 设置密码
passwd -S username       # 查看密码状态
passwd -l username       # 锁定密码认证
passwd -u username       # 解锁密码认证
passwd -d username       # 删除密码
passwd -e username       # 强制下次登录改密码

chage -l username        # 查看账号密码周期
chage -d 0 username      # 强制下次登录改密码
chage -m 1 username      # 最短使用 1 天
chage -M 40 username     # 最长使用 40 天
chage -W 7 username      # 过期前 7 天警告
chage -I 7 username      # 过期后 7 天宽限
chage -E 2026-01-01 username
```

## 7.sudo 权限下放

普通用户默认不能执行系统管理命令。需要临时授权时使用 `sudo`，配置时应使用 `visudo`，因为它会做语法检查：

```bash
visudo
```

示例：允许 `lehuang` 在本机以 root 身份免密执行 `useradd` 和 `userdel`：

```text
lehuang localhost.localdomain=(root) NOPASSWD:/usr/sbin/useradd,/usr/sbin/userdel
```

更常见的管理员授权方式是把用户加入 `wheel` 组：

```bash
usermod -aG wheel lehuang
```

# 四、文件权限管理

## 1.普通权限 r、w、x

Linux 基础权限分为读、写、执行。

| 权限 | 对文件的含义 | 对目录的含义 |
| --- | --- | --- |
| `r` | 查看文件内容 | 列出目录中的文件名 |
| `w` | 修改文件内容 | 在目录中创建、删除、重命名文件 |
| `x` | 执行文件 | 进入目录、访问目录内对象 |

查看权限：

```bash
ls -l file
ls -ld dir
```

权限示例：

```text
-rw-r--r--. 1 root root 1997 Nov 30 14:48 /etc/passwd
drwxr-xr-x. 136 root root 8192 Nov 30 21:14 /etc
```

各字段含义：

| 字段 | 含义 |
| --- | --- |
| 第 1 位 | 文件类型，如 `-` 普通文件、`d` 目录、`l` 软链接 |
| 2-10 位 | 所有者、所属组、其他人的权限 |
| `.` 或 `+` | SELinux 上下文或 ACL 标记 |
| 链接数 | 文件硬链接数或目录子目录计数 |
| 所有者 | 文件属于哪个用户 |
| 所属组 | 文件属于哪个组 |
| 大小 | 文件内容大小或目录元数据大小 |
| 时间 | 最近修改时间 |
| 文件名 | 文件或目录名称 |

## 2.文件类型

| 符号 | 类型 |
| --- | --- |
| `-` | 普通文件 |
| `d` | 目录 |
| `l` | 软链接 |
| `b` | 块设备 |
| `c` | 字符设备 |
| `s` | socket |
| `p` | 管道 |

## 3.修改归属

```bash
chown username file
chgrp groupname file
chown username:groupname file
chown -R username:groupname dir
chown --reference=ref_file target_file
```

## 4.修改权限

符号法：

```bash
chmod u+r file
chmod g-w file
chmod o= file
chmod u=rwx,g=rx,o=rx script.sh
chmod -R u=rwX,g=rX,o=rX dir
```

数字法：

| 权限 | 二进制 | 八进制 |
| --- | --- | --- |
| `rwx` | 111 | 7 |
| `rw-` | 110 | 6 |
| `r-x` | 101 | 5 |
| `r--` | 100 | 4 |
| `-wx` | 011 | 3 |
| `-w-` | 010 | 2 |
| `--x` | 001 | 1 |
| `---` | 000 | 0 |

```bash
chmod 644 file
chmod 755 dir
chmod 700 private.sh
```

目录通常需要 `x` 权限才能进入；只有 `r` 而没有 `x` 时，很多目录内容无法正常访问。

## 5.umask 默认权限

`umask` 表示系统创建文件或目录时默认保留掉的权限。

```bash
umask
umask 022
```

常见默认值：

| umask | 新文件常见权限 | 新目录常见权限 |
| --- | --- | --- |
| `022` | `644` | `755` |
| `002` | `664` | `775` |
| `077` | `600` | `700` |

永久修改通常放在 `/etc/bashrc`、`/etc/profile` 或用户自己的 Shell 配置文件中，修改后可用 `source` 立即加载：

```bash
source /etc/bashrc
source /etc/profile
```

## 6.特殊权限

| 权限 | 数字 | 常见表现 | 作用 |
| --- | --- | --- | --- |
| SUID | 4xxx | 所有者执行位显示 `s` | 执行程序时临时获得文件所有者权限 |
| SGID | 2xxx | 所属组执行位显示 `s` | 对文件表示临时获得组权限；对目录表示新文件继承目录所属组 |
| Sticky Bit | 1xxx | 其他人执行位显示 `t` | 目录中文件只能由文件所有者、目录所有者或 root 删除 |

```bash
chmod u+s file
chmod g+s dir
chmod o+t dir

chmod 4755 file
chmod 2775 shared_dir
chmod 1777 /tmp_like_dir
```

SUID 会放大权限，生产环境中应谨慎设置。常见的 Sticky Bit 示例是 `/tmp`。

## 7.ACL 权限列表

ACL 用于在传统 `u/g/o` 权限之外，对指定用户或组单独授权。

查看 ACL：

```bash
getfacl file
```

设置 ACL：

```bash
setfacl -m u:lehuang:rw file
setfacl -m g:dev:rwx dir
setfacl -x u:lehuang file
setfacl -b file
```

目录默认 ACL：

```bash
setfacl -m d:u:lehuang:rwx /mnt/pub
```

ACL 中的 `mask` 是指定用户和指定组能获得的最大权限阈值。若发现 ACL 明明授予了权限但不生效，应检查 `mask`：

```bash
setfacl -m m:rwx file
```

权限判断大致顺序：

```text
所有者 > 指定用户 ACL > 所属组/指定组 ACL > 其他人
```

## 8.attr 文件属性

`attr` 是传统权限之外的文件属性限制，对 root 也有约束效果。

```bash
lsattr file
chattr +i file     # 不允许修改、删除、重命名
chattr -i file
chattr +a file     # 只允许追加写入
chattr -a file
```

常见用途：

```bash
chattr +i /etc/passwd
chattr +a /var/log/messages
```

使用时要谨慎，否则容易造成自己无法修改配置文件。

## 9.SELinux 基础补充

RHEL 默认启用 SELinux。传统权限允许访问，不代表 SELinux 一定允许访问。查看状态：

```bash
getenforce
sestatus
```

常见模式：

| 模式 | 说明 |
| --- | --- |
| Enforcing | 强制执行策略 |
| Permissive | 只记录不拦截 |
| Disabled | 禁用 |

查看文件安全上下文：

```bash
ls -Z file
ls -Zd dir
```

恢复默认上下文：

```bash
restorecon -Rv /path
```

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

# 七、网络管理

## 1.NetworkManager

RHEL 9 使用 NetworkManager 管理网络。从 RHEL 9 开始，新连接默认以 keyfile 格式存放在：

```text
/etc/NetworkManager/system-connections/
```

旧的 `/etc/sysconfig/network-scripts/ifcfg-*` 格式仍可兼容，但不再是新配置的默认存储方式。

NetworkManager 可以管理有线、无线、动态 IP、静态 IP、物理网卡、虚拟网卡等，常用工具包括：

| 工具 | 作用 |
| --- | --- |
| `nmcli` | 命令行管理网络 |
| `nmtui` | 文本界面管理网络 |
| `nm-connection-editor` | 图形界面管理网络 |

## 2.查看网络状态

```bash
ip addr
ip route
ip link
nmcli device status
nmcli connection show
ss -tunlp
```

`ifconfig` 和 `route` 属于较旧工具，RHEL 9 中更推荐 `ip` 和 `ss`。

## 3.临时配置网络

临时配置重启网络或系统后会丢失。

```bash
ip addr add 192.168.78.210/24 dev eth1
ip link set eth1 up
ip link set eth1 down
ip route add default via 192.168.2.2
```

## 4.网卡命名规则

| 类型 | 示例 | 特点 |
| --- | --- | --- |
| 传统命名 | `eth0`、`eth1` | 依赖识别顺序，可能不稳定 |
| 可预测命名 | `ens160`、`ens192` | 依据硬件位置或固件信息，更稳定 |

RHEL/CentOS 7 之后默认使用可预测命名。若实验中想恢复 `eth0` 风格，可修改内核参数：

```bash
grubby --update-kernel ALL --args "net.ifnames=0 biosdevname=0"
reboot
```

撤销：

```bash
grubby --update-kernel ALL --remove-args "net.ifnames=0 biosdevname=0"
reboot
```

## 5.nmcli 永久配置网络

查看连接：

```bash
nmcli connection show
nmcli connection show eth1
nmcli device status
```

启动和关闭连接：

```bash
nmcli connection up eth1
nmcli connection down eth1
```

配置动态 IP：

```bash
nmcli connection add type ethernet con-name eth1 ifname eth1 ipv4.method auto autoconnect yes
```

配置静态 IP：

```bash
nmcli connection add type ethernet con-name eth1 ifname eth1 \
  ipv4.method manual \
  ipv4.addresses 192.168.78.100/24 \
  ipv4.gateway 192.168.78.2 \
  ipv4.dns 114.114.114.114 \
  autoconnect yes
```

修改已有连接：

```bash
nmcli connection modify eth1 ipv4.addresses 192.168.78.200/24
nmcli connection modify eth1 ipv4.gateway 192.168.78.2
nmcli connection modify eth1 ipv4.dns "114.114.114.114 8.8.8.8"
nmcli connection modify eth1 ipv4.method manual
nmcli connection up eth1
```

配置文件示例：

```ini
[connection]
id=eth1
type=ethernet
interface-name=eth1

[ipv4]
address1=192.168.78.200/24,192.168.78.2
dns=114.114.114.114;
method=manual

[ipv6]
method=auto
```

手工修改 `.nmconnection` 文件后，通常需要：

```bash
chmod 600 /etc/NetworkManager/system-connections/*.nmconnection
nmcli connection reload
nmcli connection up eth1
```

## 6.ping 与基础排错

```bash
ping -c 4 www.baidu.com
ping -c 4 8.8.8.8
ping -s 1000 -c 4 192.168.78.2
ping -t 64 -c 4 192.168.78.2
```

`ping` 输出中的常见字段：

| 字段 | 含义 |
| --- | --- |
| bytes | ICMP 数据大小 |
| time | 响应耗时，越小越快 |
| ttl | IP 包生存时间，每经过一跳通常减 1 |

TTL 不是 DNS 记录存在时间。DNS 记录也有 TTL，但 `ping` 输出中的 TTL 是 IP 报文的 TTL。

基础排错顺序：

```bash
ip addr                         # 是否有 IP
ip route                        # 是否有默认网关
ping -c 4 gateway               # 网关是否通
ping -c 4 8.8.8.8               # 外网 IP 是否通
ping -c 4 www.baidu.com         # DNS 是否正常
cat /etc/resolv.conf            # DNS 配置
nmcli device status             # NM 管理状态
```

## 7.域名解析

静态解析：

```bash
vim /etc/hosts
```

示例：

```text
154.19.232.119 www.example.com
```

DNS 解析可通过 NetworkManager 配置：

```bash
nmcli connection modify eth1 ipv4.dns "114.114.114.114 8.8.8.8"
nmcli connection up eth1
```

排查 DNS：

```bash
getent hosts www.baidu.com
resolvectl status
```

## 8.wget 和 curl

`wget` 适合下载文件，`curl` 更适合调试 HTTP 请求、上传下载和接口测试。

```bash
wget -P /tmp https://example.com/file.tar.gz
wget -c https://example.com/big.iso
wget -r -p https://example.com/

curl -o index.html https://www.baidu.com
curl -I https://www.baidu.com
curl -L https://example.com
```

## 9.VMware 三种网络模式

VMware 常见网络模式有桥接、NAT、仅主机三种。

![VMware 虚拟网卡概览](https://cdn.jsdelivr.net/gh/Lehuang0212/blog-images/images/20251214211710548.png)

### Bridged 桥接模式

桥接模式让虚拟机像局域网中的一台真实主机一样接入网络。虚拟机 IP 应与物理主机在同一网段，网关和 DNS 通常与物理主机一致。

![VMware 桥接模式](https://cdn.jsdelivr.net/gh/Lehuang0212/blog-images/images/20251214212020750.png)

适用场景：

- 虚拟机需要被局域网其他主机直接访问。
- 局域网 IP 资源充足。
- 网络环境允许新增主机接入。

注意事项：

- 虚拟交换机应桥接到当前真正联网的物理网卡。
- 不建议在复杂网络中长期使用“自动桥接”。

### NAT 地址转换模式

NAT 模式通过 VMware 的虚拟 NAT 设备和虚拟 DHCP 服务让虚拟机访问外网。外部网络通常不能直接访问虚拟机，除非配置端口转发。

![VMware NAT 模式](https://cdn.jsdelivr.net/gh/Lehuang0212/blog-images/images/20251214212208551.png)

适用场景：

- IP 资源紧张。
- 只要求虚拟机能访问外网。
- 实验环境不希望暴露虚拟机。

### Host-Only 仅主机模式

Host-Only 模式把虚拟机限制在主机和虚拟机组成的私有网络中。默认不能访问外网，适合封闭实验环境、内部服务测试、安全隔离练习。

| 模式 | 虚拟机访问外网 | 主机访问虚拟机 | 局域网其他机器访问虚拟机 |
| --- | --- | --- | --- |
| Bridged | 可以 | 可以 | 可以 |
| NAT | 可以 | 可以 | 通常不可以 |
| Host-Only | 默认不可以 | 可以 | 不可以 |

# 八、文件传输、归档与压缩

## 1.sshd 服务

Linux 主机之间常用 SSH 协议进行远程登录和文件传输。先确认 `sshd` 服务：

```bash
systemctl status sshd
systemctl enable --now sshd
```

如果防火墙开启，需要放行 SSH：

```bash
firewall-cmd --add-service=ssh --permanent
firewall-cmd --reload
```

不建议为了传文件直接关闭防火墙。实验中若临时关闭，也要知道它的影响：

```bash
systemctl disable --now firewalld
```

## 2.scp

`scp` 基于 SSH，用法简单，适合临时复制文件。

```bash
scp file root@192.168.78.129:/root/Desktop/
scp -r dir root@192.168.78.129:/root/Desktop/
scp root@192.168.78.129:/root/Desktop/file /tmp/
```

常用参数：

| 参数 | 作用 |
| --- | --- |
| `-r` | 复制目录 |
| `-q` | 安静模式，不显示进度 |
| `-P` | 指定 SSH 端口，注意是大写 |

## 3.rsync

`rsync` 支持增量同步，适合重复传输、目录同步和备份。

```bash
rsync -av file root@192.168.78.129:/root/Desktop/
rsync -av dir/ root@192.168.78.129:/root/Desktop/dir/
rsync -av root@192.168.78.129:/mnt/ /mnt/
```

常用参数：

| 参数 | 作用 |
| --- | --- |
| `-a` | 归档模式，包含递归、权限、时间戳、链接等 |
| `-v` | 显示过程 |
| `-z` | 传输时压缩 |
| `--delete` | 删除目标端多余文件，使两端一致 |
| `--progress` | 显示传输进度 |

注意源路径结尾 `/` 的区别：

```bash
rsync -av /mnt root@host:/backup/      # 把 mnt 目录本身复制到 /backup/
rsync -av /mnt/ root@host:/backup/     # 只复制 /mnt/ 中的内容
```

## 4.测试文件生成

```bash
truncate -s 500M file
dd if=/dev/zero of=file bs=1M count=500
```

`truncate` 速度快，常用于创建稀疏文件；`dd` 会实际写入数据，更适合模拟真实写入。

## 5.tar 归档

归档是把多个文件或目录打包成一个文件，不一定压缩。

```bash
tar cf etc.tar /etc
tar tf etc.tar
tar rf etc.tar file
tar xf etc.tar
tar xf etc.tar -C /mnt
tar --delete -f etc.tar file
tar --get -f etc.tar file
```

常用参数：

| 参数 | 作用 |
| --- | --- |
| `c` | 创建归档 |
| `x` | 解归档 |
| `t` | 查看归档内容 |
| `f` | 指定归档文件名 |
| `v` | 显示过程 |
| `r` | 向归档中追加文件 |
| `-C` | 指定解包目录 |

## 6.压缩格式

| 格式 | 特点 |
| --- | --- |
| `.gz` | 速度快，压缩率一般 |
| `.bz2` | 压缩率较高，速度较慢 |
| `.xz` | 压缩率高，速度更慢 |
| `.zip` | 跨平台友好 |

tar 归档压缩：

```bash
tar zcf file.tar.gz dir
tar jcf file.tar.bz2 dir
tar Jcf file.tar.xz dir

tar zxf file.tar.gz
tar jxf file.tar.bz2
tar Jxf file.tar.xz
```

# 九、防火墙基础补充

RHEL 9 默认使用 firewalld 管理防火墙。

```bash
systemctl status firewalld
firewall-cmd --state
firewall-cmd --get-default-zone
firewall-cmd --list-all
```

放行服务和端口：

```bash
firewall-cmd --add-service=http --permanent
firewall-cmd --add-port=8080/tcp --permanent
firewall-cmd --reload
```

移除规则：

```bash
firewall-cmd --remove-service=http --permanent
firewall-cmd --remove-port=8080/tcp --permanent
firewall-cmd --reload
```

排查服务访问问题时，顺序通常是：

```bash
systemctl status 服务名
ss -tunlp
firewall-cmd --list-all
getenforce
journalctl -u 服务名 -xe
```

# 十、常用排障清单

## 1.系统版本和内核

```bash
cat /etc/redhat-release
uname -r
hostnamectl
```

## 2.资源占用

```bash
uptime
top
free -h
df -h
du -sh /var/log
```

## 3.服务异常

```bash
systemctl status service
journalctl -u service -xe
systemctl restart service
```

## 4.网络不通

```bash
ip addr
ip route
nmcli device status
ping -c 4 网关
ping -c 4 8.8.8.8
ping -c 4 域名
ss -tunlp
```

## 5.权限异常

```bash
ls -l file
namei -l /path/to/file
getfacl file
lsattr file
getenforce
ls -Z file
```

## 6.软件安装失败

```bash
dnf repolist
dnf clean all
dnf makecache
dnf provides command_name
dnf history
```

# 十一、基础总结

Linux 基础不是零散命令的堆砌，而是一套围绕“资源管理”的体系：用户和权限决定谁能访问资源，进程和服务决定程序如何运行，软件管理决定系统能力如何扩展，网络和文件传输决定主机如何与外界协作。

学习时建议按以下顺序练习：

1. 先熟悉路径、文件、帮助命令和文本编辑。
2. 再掌握用户、组、权限、ACL、SELinux 的关系。
3. 然后学习进程、服务、日志和软件包管理。
4. 最后把网络配置、SSH、传输、归档和防火墙串起来做完整实验。

当遇到问题时，不要只记某个命令的结果，而要追问：这个问题属于身份、权限、进程、服务、软件、网络、存储中的哪一类。分类清楚后，Linux 的排障会轻松很多。
