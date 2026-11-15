---
title: "Linux 基本操作（一）：文件、用户与权限"
date: "2026-06-15T00:00:00+08:00"
tags:
  - Linux
  - RHEL
  - 文件系统
  - 权限
description: "以 RHEL 9.6 为例梳理 Linux 入门框架、文件系统、常用文件命令、用户与用户组以及文件权限，结合命令示例说明日常操作与权限管理的关键概念。"
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

| 目录              | 作用                |
| --------------- | ----------------- |
| `/`             | 根目录，所有目录的起点       |
| `/boot`         | 内核、引导加载器相关文件      |
| `/etc`          | 系统和服务配置文件         |
| `/home`         | 普通用户家目录           |
| `/root`         | root 用户家目录        |
| `/usr`          | 系统程序、库、文档等主要软件资源  |
| `/var`          | 经常变化的数据，如日志、缓存、邮件 |
| `/tmp`          | 临时文件目录            |
| `/dev`          | 设备文件              |
| `/proc`         | 内核和进程的虚拟文件系统      |
| `/run`          | 系统启动后生成的运行时数据     |
| `/mnt`、`/media` | 临时挂载点             |

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

下一篇：[Linux 基本操作（二）：进程、服务与软件](/articles/Linux/Linux基本操作-进程服务与软件/)。
