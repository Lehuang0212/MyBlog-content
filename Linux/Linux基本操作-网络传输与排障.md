---
title: "Linux 基本操作（三）：网络、传输与排障"
date: "2026-06-15T00:00:00+08:00"
tags:
  - Linux
  - RHEL
  - 网络
  - 排障
description: "接续 Linux 基本操作系列，介绍网络配置与诊断、文件传输、归档压缩、防火墙基础及常用排障清单，帮助把基础命令串成实际运维流程。"
---

上一篇：[Linux 基本操作（二）：进程、服务与软件](/articles/Linux/Linux基本操作-进程服务与软件/)。

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

![VMware 虚拟网卡概览](https://raw.githubusercontent.com/Lehuang0212/Myblog-images/main/images/linux/20251214211710548.png)

### Bridged 桥接模式

桥接模式让虚拟机像局域网中的一台真实主机一样接入网络。虚拟机 IP 应与物理主机在同一网段，网关和 DNS 通常与物理主机一致。

![VMware 桥接模式](https://raw.githubusercontent.com/Lehuang0212/Myblog-images/main/images/linux/20251214212020750.png)

适用场景：

- 虚拟机需要被局域网其他主机直接访问。
- 局域网 IP 资源充足。
- 网络环境允许新增主机接入。

注意事项：

- 虚拟交换机应桥接到当前真正联网的物理网卡。
- 不建议在复杂网络中长期使用“自动桥接”。

### NAT 地址转换模式

NAT 模式通过 VMware 的虚拟 NAT 设备和虚拟 DHCP 服务让虚拟机访问外网。外部网络通常不能直接访问虚拟机，除非配置端口转发。

![VMware NAT 模式](https://raw.githubusercontent.com/Lehuang0212/Myblog-images/main/images/linux/20251214212208551.png)

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
