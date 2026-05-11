---
title: Linux的网络管理
date: 2025-12-13 18:21:10
tags:
---
本篇文章将介绍Linux的网络管理基础内容。
<!-- more -->

# 前言
在RHEL9上，使用NM进行网络配置，ifcfg （也称为 文件）将不再是网络配置文件的主存储。虽然 ifcfg 样式仍然可用，但它不再是 NetworkManager 存储新网络配置文件的默认位置。从 RHEL 9.0 开始，RHEL 以 key-file 格式在 /etc/NetworkManager/system-connections/ 中存储新的网络配置。配置以旧格式存储在 /etc/sysconfig/network-scripts/ 中的连接仍然可以正常工作。对现有配置集的修改会继续更新旧的文件。

NetworkManager是2004年Red Hat启动的项目，旨在能够让Linux用户更轻松地处理现代网络需求，能自动发现网卡并配置ip地址。

## NetworkManager的特点
NM能管理各种网络：
    有线网卡、无线网卡
    动态ip、静态ip
    以太网、非以太网
    物理网卡、虚拟网卡
**工具齐全**：命令行、文本界面、图形界面、web
**广纳天地**：管理各种网络，有线、无线、物理、虚拟
**参数丰富**：多达200多项配置参数（包括ethtool参数）
**一统江湖**：RedHat系、Suse系、Debian/Ubuntu系，均支持
**大势所趋**：下一个大版本的rhel只能通过NM管理网络

# 一、配置网络
## 1.临时配置网络
在Linux中查看网络的命令行指令ip和ifconfig，两者都可以用来查看Linux的网络信息，同时也可以进行编辑，但是需要注意，二者去对网络进行配置修改都只是暂时的，重启网络服务、重启系统，或者网卡重新激活后，配置会自动丢失，恢复到系统默认的网络设置。
```bash
# 查看网络信息有两种方法
ip a
ifconfig
# 查看过网络信息之后，我们可以使用watch命令进行监控
watch -n 1 "ip a"
watch -n 1 "ifconfig"
# 接下来我们可以进行临时修改网络的操作
# 给网卡设临时 IP
ip addr add 192.168.78.210/24 dev eth1
ifconfig eth1 192.168.78.210 netmask 255.255.255.0
# 临时启用/禁用网卡
ip link set eth1 up/down
ifconfig eth1 up/down
# 临时添加默认网关(gateway)
ip route add default via 192.168.2.2
route add default gw 192.168.2.2
```
## 2.永久配置网络
首先需要先了解一下网卡的命名规则:主要有两种，eth和ens。
|特性|传统 eth 命名（eth0/eth1）|可预测 ens 命名（ens160/ens192）|
|---|---|---|
|命名依据|系统识别网卡的先后顺序|网卡的物理属性 / 硬件信息|
|命名规则|第一个识别的 = eth0，第二个 = eth1|ens 含义：<br>- en = 以太网<br>- s = 主板内置卡槽<br>- 160 = 卡槽编号|
|稳定性|极不稳定：重启 / 插拔网卡可能变号|绝对稳定：命名和物理位置绑定|
|适用场景|单网卡 / 测试环境 / 习惯旧命名|多网卡 / 生产环境 / 需要固定命名|
|默认启用版本|RHEL/CentOS 6 及以前默认|RHEL/CentOS 7+ 默认|

在RHEL/CentOS 7+系统中，如果要将命名规则重新修改为eth，需要禁用 predictable network interface names（可预测网络接口命名），从而恢复传统的 eth0/eth1 命名方式。
可以使用以下命令:
```bash
grubby --update-kernel ALL --args net.ifnames=0
#一般来说执行以上指令重启后就可以恢复eth命名规则

#要彻底恢复 eth 命名，需要同时添加两个参数：
grubby --update-kernel ALL --args "net.ifnames=0 biosdevname=0"
# net.ifnames=0：关闭 systemd 的可预测命名规则
# biosdevname=0：关闭 Dell 硬件的 biosdevname 命名规则

#若后续需要撤销该修改，可使用：
grubby --update-kernel ALL --remove-args "net.ifnames=0 biosdevname=0"
```

接下来介绍永久配置网络的方式，使用nmcli命令
```bash
# 查看网卡信息
nmcli connection show
# 查看指定网卡信息
nmcli connection show eth1
# 查看已有设备的连接状态
nmcli connection dev status # 按q退出
# 激活和关闭网络连接
nmcli connection up/down
```
在进行添加/删除/修改网络连接时，有许多方式可以永久的进行增删改。
```bash
# 对于图形模式(进行GUI操作)的增删改，有以下方式
nm-connection-editor # 输入后，使用弹出的管理器界面进行增删改

# 对于无图形模式，也可以进行增删改
init 3
systemctl isolate multi-user.target  #进入无图形模式
nmtui # 输入后，在对应界面使用键盘交互操作进行增删改
```
nm-connection-editor:
![](https://cdn.jsdelivr.net/gh/Lehuang0212/blog-images/images/20251214190643746.png)
nmtui：
![](https://cdn.jsdelivr.net/gh/Lehuang0212/blog-images/images/20251214190519783.png)
用命令的形式对网络进行增删改有两种方式
```bash
# 纯命令操作
nmcli connection add ...
# 示例：
nmcli connection add type ethernet con-name eth1 ifname eth1 ipv4.method manual ipv4.address '192.168.78.100/24' autoconnect yes # 手动IP
nmcli connection add type ethernet con-name eth1 ifname eth1 ipv4.method auto #自动IP
# 纯命令操作时，add后面可操作诸多信息，按tab补全可以查看，也可以配置网关和DNS
nmcli connection add type ethernet con-name eth1 ifname eth1 ipv4.method manual ipv4.address '192.168.78.100/24' ipv4.gateway '192.168.2.2' ipv4.dns '114.114.114.114'

# 通过修改配置文件进行操作
# 文件位于/etc/NetworkManager/system-connections
# /etc/NetworkManager/system-connections中储存着网络信息
# 示例：
vim /etc/NetworkManager/system-connections/eth1.nmconnection
# 内容：
[connection]
id=eth1
uuid=4b67e760-490a-472b-930a-7e64dcf623b4
type=ethernet
interface-name=eth1

[ethernet]

[ipv4]
address1=192.168.78.200/24
method=manual

[ipv6]
addr-gen-mode=default
method=auto

[proxy]

# 只需要在对应位置加上对应信息即可进行更改，增删网络也可以通过增删.nmconnection文件来实现
```
修改配置文件：
![](https://cdn.jsdelivr.net/gh/Lehuang0212/blog-images/images/20251214202810221.png)

## 3.利用ping命令
ping命令可以用来检测网络的连通情况和分析网络速度，根据域名得到服务器 IP，根据 ping 返回的 TTL 值来判断对方所使用的操作系统及数据包经过路由器数量。
```bash
# ping命令的使用
ping [options] <destination>
options:
    -c 数字  #用于设定本命令发出的ICMP消息包的数量，若无此选项，则会无限 次发送消息包直到用户按[Ctrl+C]组合键才终止命令。
    -s 字节数  #设置ping命令发出的消息包的大小，默认发送的测试数据大小为56字节；自动添加8字节的ICMP协议头后,显示的是64字节；再添加20字节的IP协议头，则显示的为84字节。最大设置值为65507B
    -i  #设定前后两次发送ICMP消息包之间的时间间隔，无此选项时，默认时间间隔为1秒。为了保障本机和目标主机的安全，一般不要小于0.2秒
    -t  #设置存活时间TTL(Time To Live)
    -w  #设定执行时间
# 示例：
ping -c1 www.baidu.com
# 输出显示：
PING wwwbaidu.com (154.19.232.119) 56(84) bytes of data.
64 bytes from 154.19.232.119 (154.19.232.119): icmp_seq=1 ttl=128 time=160 ms
# 显示内容分析
bytes值:数据包的大小
time值:响应时间，时间越小，速度越快
TTL值:Time To Live, 表示 DNS 记录在 DNS 服务器上存在的时间，它是 IP 协议的一个值，告诉路由器该数据包何时需要被丢弃。通过TTL 值判断目标系统类型：默认况下，Linux 系统的 TTL 值为 64 或 255，win11 系统值为644，UNIX 主机的 TTL 值为 255
```

# 二、网络的简单使用
## 1.通过域名访问主机
### (1)静态解析
可以通过配置/etc/hosts文件实现域名解析
```bash
vim /etc/hosts
#示例(在末尾加上IP+域名)：
154.19.232.119 www.baidu.com
```
### (2)DNS域名解析
DNS 是分布式的 “域名 - IP 翻译系统”，通过 “本地缓存 + 分层查询” 实现高效、可靠的域名到 IP 映射，是互联网访问的基础（无 DNS 则需直接输入 IP 才能访问主机）。
可以使用nmcli命令在添加网络的时候给其添加DNS地址或者在nm-connection-editor界面和nmtui界面进行配置。同样的，配置DNS也可以通过修改配置文件/etc/NetworkManager/system-conncetions/.nmconnection文件来进行配置。
![](https://cdn.jsdelivr.net/gh/Lehuang0212/blog-images/images/20251214210410568.png)

## 2.从网站下载文件
wget命令用于在终端命令行里下载网络文件，英文全称为：“web get”。
curl命令是一个网络工具，其主要作用是通过http、https、ftp等方式下载/上传文件。
```bash
wget [OPTION]... [URL]...
OPTION:
    -P  #定义下载文件的保存路径，目录不存在时自动创建；
    -t  #网络故障时的重试次数，默认 20 次，t 0表示无限尝试
    -b  #启动后 wget 转入后台运行，不占用当前终端
    -c  #针对未下载完成的文件，继续下载（而非重新下载）
    -p  #下载目标网页的所有关联资源（图片、CSS、JS、视频、字体等）
    -r  #遍历目标 URL 的目录层级，递归下载子页面 / 子目录文件
# 示例下载qq软件：
wget ~/Desktop https://dldir1.qq.com/qqfile/qq/QQNT/2355235c/linuxqq_3.1.1-11223_x86_64.rpm

curl [options...] <url>
# 示例：
curl www.baidu.com -o index.html
```
# 三、VMware三种网络模式
## 概述
vmware提供三种网络工作模式：Bridged（桥接模式）、NAT（网络地址转换模式）、Host-Only（仅主机模式）
![](https://cdn.jsdelivr.net/gh/Lehuang0212/blog-images/images/20251214211710548.png)
vmware->编辑菜单->虚拟网络编辑器
![](https://cdn.jsdelivr.net/gh/Lehuang0212/blog-images/images/20251214211848233.png)
## Bridged（桥接模式）
桥接模式就是将主机网卡与虚拟机虚拟的网卡利用虚拟网桥进行通信。
在桥接的作用下，类似于把物理主机虚拟为一个交换机，所有桥接设置的虚拟机连接到这个交换机的一个接口上，物理主机也同样插在这个交换机当中，所以所有桥接下的网卡与网卡都是交换模式的，相互可以访问而不干扰。
在桥接模式下，虚拟机ip地址需要与主机在同一个网段，如果需要联网，则网关与DNS需要与主机网卡一致。
![](https://cdn.jsdelivr.net/gh/Lehuang0212/blog-images/images/20251214212020750.png)
**注意**
**·**虚拟交换机必须桥接到本机电脑使用的网卡，不要选自动。
**·**虚拟机的ip跟本机同一个网段，网关相同。
**·**桥接模式配置简单，如果网络环境中ip资源很缺少或对ip管理比较严格，则桥接模式就不太适合。
## NAT（地址转换模式）
NAT模式借助虚拟NAT设备和虚拟DHCP服务器，使得虚拟机可以联网。其网络结构如下图：
![](https://cdn.jsdelivr.net/gh/Lehuang0212/blog-images/images/20251214212208551.png)
**注意**
**·**NAT模式是利用虚拟的NAT设备以及虚拟DHCP服务器来使虚拟机连接外网，VMware Network Adapter VMnet8虚拟网卡是用来与虚拟机通信的
**·**如果你的网络ip资源紧缺，但是你又希望你的虚拟机能够联网，这时候NAT模式是最好的选择
## Host-Only（仅主机模式）
Host-Only模式其实就是NAT模式去除了虚拟NAT设备，然后使用VMnet1虚拟网卡连接VMnet1虚拟交换机来与虚拟机通信的。
Host-Only模式将虚拟机与外网隔开，使得虚拟机成为一个独立的系统，只与主机相互通讯。

