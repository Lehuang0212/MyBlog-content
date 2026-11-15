---
title: "RHCSA 习题解析"
date: "2026-08-02T16:23:38+08:00"
tags:
  - Linux
  - RHEL
  - RHCSA
description: "按 servera 与 serverb 的练习题整理 RHCSA 配置步骤，涵盖网络、软件仓库、SELinux、用户权限、存储、容器与系统服务，并对关键命令作解释。"
---

# servera试题
## 1、主机名网络配置和SSH链接
### 题目：
```text
配置网络，要求如下 
1. 主机名： servera.lab.example.com 
2. IP 地址：172.25.250.10/24 
3. 网关：172.25.250.254 
4. DNS：172.25.250.220
```
### 解答：
```bash
# 查看网卡和对应名
nmcli connection show 
# 配置ipv4地址的分配模式为manual（手动）,并且指明了地址、网关、dns
nmcli connection modify NAME ipv4.method manual ipv4.addresses 172.25.250.10/24 ipv4.gateway 172.25.250.254 ipv4.dns 172.25.250.220
# 网络配置完成后需要重载nmcli connection reload
nmcli connection up NAME
# 设置主机名hostnamectl set-hostname servera.lab.example.com
# 重载bash，检查名字是否更改bash
# 编辑ssh配置文件，将PermitRootLogin改为yes
vim /etc/ssh/sshd_config
PermitRootLogin yes
# 后续ssh链接
ssh root@172.25.250.10
```
## 2、YUM仓库配置
### 题目：
```text
配置系统的软件仓库YUM 的两个存储库的地址分别是
BaseOS:http://content.example.com/rhel9.0/x86_64/dvd/BaseOS
AppStream:http://content.example.com/rhel9.0/x86_64/dvd/AppStream
```
### 仓库文件参数详解：

| **参数名称**              | **是否必填** | **说明与配置规范**                                                                        |
| --------------------- | -------- | ---------------------------------------------------------------------------------- |
| **`[repository_id]`** | **必填**   | 仓库的*唯一标识符*（中括号内）。同一个系统内不能重复，**不能包含空格**。                                            |
| **`name`**            | **必填**   | 仓库的描述性名称（人类可读）。可以是任意说明文字，用于在终端列表中识别该源。                                             |
| **`baseurl`**         | **二选一**  | 软件包及元数据（`repodata` 目录所在位置）的**绝对路径或URL**。支持`file://`、`http://`、`https://`、`ftp://`。 |
| **`mirrorlist`**      | **二选一**  | 指向一个包含多个镜像服务器 URL 的文本文件地址（与 `baseurl` 互斥，通常用于公网镜像源服务）。                             |
| **`enabled`**         | 选填       | 是否启用该仓库：<br>•`1` 或`true`：**启用**（默认值）<br>•`0` 或`false`：**禁用**                         |
| **`gpgcheck`**        | 选填       | 是否开启RPM 包的 GPG 数字签名校验：<br>•`1`：*开启*（防止软件包被篡改，推荐生产环境开启）<br>•`0`：**关闭**（测试或本地源常用）     |
| **`gpgkey`**          | 条件必填     | GPG 公钥文件的路径或 URL。*仅当 `gpgcheck=1` 时必须配置*。                                          |
### 解答：
```bash
# 仓库文件路径为 /etc/yum.repos.d/，新建以 .repo 结尾的文件
vim /etc/yum.repos.d/rhel.repo
# 仓库文件格式如下，注意不要带空格，路径名字严格大小写对齐
[BaseOS]
name=BaseOS
baseurl=http://content.example.com/rhel9.0/x86_64/dvd/BaseOS
gpgcheck=0
[AppStream]
name=AppStream
baseurl=http://content.example.com/rhel9.0/x86_64/dvd/AppStream
gpgcheck=0
```
在配置`baseurl` 或`gpgkey` 时，可以使用 Yum/DNF 提供的系统内置变量，使配置文件具备跨版本或跨架构的通用性：
- **`$releasever`**：当前系统的**主要版本号**（例如RHEL 9 中该变量会被自动替换为`9` 或`9.0`）。- **`$basearch`**：系统的**硬件架构**（例如`x86_64`、`aarch64`）。- **`$infra`**：系统基础设施类型（如 `stock`）。

## 3、配置SELinux
### 题目：
```text
非标准端口82 上运行的 WEB 服务器在提供内容时遇到问题。根据需要调试并解决问题，并使其满足以下条件: 
1. web 服务器能够提供/var/www/html 中所有现有的 html 文件(注意：不要删除或改动现有的文件内容) 
2. Web 服务器通过 82 端口访问 
3. Web 服务器在系统启动时自动启动
```
### SELinux 核心机制与安全上下文格式

SELinux（Security-Enhanced Linux）是一种强制访问控制（MAC）安全机制。SELinux 为系统中的每个进程和文件（包括目录、端口等）都打上了一个安全上下文（Security Context）标签。
安全上下文的标准格式为：

`user:role:type:level`

例：`system_u:object_r:httpd_sys_content_t:s0`

- **User（用户）**：例如`system_u`（系统用户）、`unconfined_u`（非受限用户）。- **Role（角色）**：例如`object_r`（文件对象角色）、`system_r`（进程角色）。- **Type（类型）- 最核心**：在 Targeted 模式下，SELinux 主要通过类型匹配（TE，Type Enforcement）进行权限控制。    - 进程类型通常被称为*域（Domain）*，如 `httpd_t`。    - 文件/端口类型被称为*类型（Type）*，如 `httpd_sys_content_t`、`http_port_t`。    - **核心规则**：只有当 `httpd_t` 进程被策略明确允许访问`httpd_sys_content_t` 文件和`http_port_t` 端口时，操作才会被批准。

### 管理网络端口：`semanage port`命令详解：
```bash
semanage port {-a|-d|-m|-l} -t 端口类型 -p 协议 
端口号常用参数：
-a --add        添加新端口定义。
-d, --delete    删除指定的端口定义。
-m, --modify    修改现有的端口定义。
-l, --list      列出所有端口映射规则。
-t, --type      指定 SELinux 端口类型（如 http_port_t, ssh_port_t）。
-p, --proto     指定网络协议（tcp 或udp）。
```
### 管理文件上下文：`semanage fcontext`命令详解：
```bash
semanage fcontext {-a|-d|-m|-l} -t 文件类型 "文件路径正则"
-a：添加策略映射。
-d：删除策略映射。
-t：指定目标文件类型（如`httpd_sys_content_t`）。
"(.*)?" 或"(.*)"：正则表达式，表示匹配目录本身及其内部所有子文件/子目录。
```
### SELinux 常用辅助命令：
```bash
## 查看当前selinux运行模式
getenforce
## 临时切换selinux运行模式(0为Permissive宽容，1为Enforcing强制)
setenforce
## 查看文件或目录的selinux上下文
ls -Z
## 根据策略库恢复文件正确的上下文
restorecon
restorecon -Rv /path # -R是递归，-v显示过程，path是路径# 临时修改文件上下文chcon
## 示例
chcon -R -t httpd_sys_content_t /var/www/html/
```
### 解答：
```bash
# 添加/修改 SELinux 允许的端口
semanage port -a -t http_port_t -p tcp 82
# 修复文件 SELinux 上下文
semanage fcontext -a -t httpd_sys_content_t "/var/www/html(/.*)?"
# 刷新上下文
restorecon -Rv /var/www/html/
# 配置防火墙放行端口
firewall-cmd --permanent --add-port=82/tcp
firewall-cmd --reload
# 设置服务开机自启并启动
systemctl enable httpd --now
# 重启服务测试
systemctl restart httpd
```
## 4、创建用户
### 题目：
```text
创建下列用户、用户组，并按要求完成设置 
1. 组名为sysmgrs 
2. natasha 用户的附属组是sysmgrs 
3. harry 用户的附属组是sysmgrs 
4. john 用户的shell 是非交互式shell，且不是 sysmgrs 组的成员 
5. natasha、harry、john 的密码是 redhat
```
### 所需命令：
```bash
groupadd    创建组
useradd     创建用户
	-g    指定用户的主要组
	-G    指定用户的附属组
usermod     修改已存在的用户
	-G    修改附属组	-aG   追加附属组# 修改natasha用户，将其加入sysmgrs组usermod -aG sysmgrs natasha

## 修改密码
echo "password" | passwd --stdin 用户名
/sbin 是软链接指向 /usr/sbin
/sbin/nologin 非交互式shell
```
### 解答：
```bash
# 创建sysmgrs组groupadd sysmgrs
# 创建natasha用户，并将其加入sysmgrs组useradd -G sysmgrs natasha
# 创建harry用户，并将其加入sysmgrs组useradd -G sysmgrs harry
# 创建john用户，指定其shell为非交互式useradd -s /sbin/nologin john
# 修改密码为redhat
echo redhat | passwd --stdin natasha
echo redhat | passwd --stdin harry
echo redhat | passwd --stdin john
```
## 5、配置cron计划任务
### 题目：
```text
该作业每隔两分钟执行以下命令：
logger “hello rhcsa” 以用户natasha 身份运行
```
### `cron` 时间语法详解

`crontab` 文件中的时间共有 **5 个字段**（用空格分隔），格式如下：
`分钟 小时 日 月 星期 要执行的命令`

|**字段**|**取值范围**|**本题配置**|**含义**|
|---|---|---|---|
|**分钟 (Minute)**|`0 - 59`|**`*/2`**|每隔 2 分钟执行一次|
|**小时 (Hour)**|`0 - 23`|**`*`**|每小时|
|**日(Day of month)**|`1 - 31`|**`*`**|每天|
|**月(Month)**|`1 - 12`|**`*`**|每个月|
|**星期 (Day of week)**|`0 - 7`（0 和 7 都代表周日）|**`*`**|每周的每一天|

### 符号规则说明：
- **`*`**：代表“任意”或“每”（如小时字段写 `*` 代表每小时）。- **`*/n`**：代表“每隔n 个单位时间”（如`*/2` 在分钟字段代表每 2 分钟，在小时字段代表每2 小时）。- **`,`**：代表分隔离散时间点（如 `1,3,5` 在分钟字段代表第 1、3、5 分钟）。- **`-`**：代表连续时间段（如 `1-5` 在星期字段代表周一到周五）。

### `crontab` 命令常用选项
- **`-e`（edit）**：编辑当前用户（或`-u` 指定用户）的定时任务文件。- **`-l`（list）**：列出当前用户（或`-u` 指定用户）的定时任务。- **`-r`（remove）**：清空并删除指定用户的全部定时任务- **`-u <username>`**：指定要操作的目标用户。仅 `root` 用户有权限使用`-u` 管理其他用户的任务。
### 解答：
```bash
# 添加定时任务，指定用户natasha
crontab -e -u natasha
# 定时任务内容
*/2 * * * * /usr/bin/logger “hello rhcsa”
```
## 6、按要求创建目录
### 题目：
```text
1. /home/managers 目录属于 sysmgrs 统
2. 此目录可以被 sysmgrs 的组成员读取、写入和访问，但是其他任何用户不具备这些权限。(不包括root 用户) 
3. 在home/managers 目录中创建的文件的所属组自动变成 sysmgrs 统
```
### Linux 权限基础回顾

理解 `chmod` 前，必须先理解Linux 的**三类用户**和**三种基本权限**。
### 三类身份（Target）
- **`u` (User / Owner)**：文件所有者（属主）
- **`g` (Group)**：文件所属组（属组）
- **`o` (Others)**：其他用户- **`a` (All)**：代表以上所有人
### 三种基本权限（Mode）
|**权限字符**|**权限名称**|**对应数字**|**对文件的作用**|**对目录的作用**|
|---|---|---|---|---|
|**`r`**|读（Read）|**4**|查看文件内容（如 `cat`）|查看目录下的文件名列表（如`ls`）|
|**`w`**|写（Write）|**2**|修改文件内容（如 `vim`）|在目录下新建、删除、重命名文件|
|**`x`**|执行（Execute）|**1**|将文件当作程序脚本运行|进入该目录（如`cd`），以及读取目录下文件的属性|

### 四位数字与特殊权限位
（SUID, SGID, Sticky Bit：当在八进制数字前加上第4 位（首位）时，设置的是Linux 的特殊权限。

```bash
chmod 2770 /home/managers
```

|**特殊权限**|**对应首位数值**|**符号表示**|**作用对象**|**核心作用**|
|---|---|---|---|---|
|**SUID** (Set UID)|**`4`**|`u+s`|可执行二进制文件|任何用户执行该命令时：**临时获取该文件属主的身份**运行（如 `/usr/bin/passwd`）。|
|**SGID** (Set GID)|**`2`**|`g+s`|目录（或文件）|**用于目录**：在该目录下新建的文件，**强制继承该目录的属组**（团队协作目录必备）。|
|**Sticky Bit** (防删除)|**`1`**|`o+t` 或`+t`|共享目录|**用于目录**：任何人都有写权限，但*只能删除/修改自己创建的文件*，无法删除别人的文件（如 `/tmp`）。|
#### 权限数字 `2770` 拆解：
- **第一位`2`**：代表**SGID 特殊权限**（SUID=4, SGID=2, Sticky=1）。
- **第二位`7`**（所有者`root`）：`r(4) + w(2) + x(1) = 7`
- **第三位`7`**（所属组 `sysmgrs`）：`r(4) + w(2) + x(1) = 7`
- **第四位`0`**（其他用户`others`）：`无权限= 0`
### 解答：
```bash
# 先创建目录mkdir /home/managers
# 修改所有组
chgrp sysmgrs /home/managers
# 修改目录权限，两种写法皆可chmod g=rwxs,o=- /home/managers
chmod 2770 /home/managers
# 查看权限是否正确
ll -d /home/managers
# 前半部分如下则正确drwxrws---. 2 root sysmgrs
```
## 7、配置NTP时间服务器
### 题目：
```text
配置 servera 作为 NTP 的客户端，跟时间服务器172.25.250.254 同步时间
```
### 配置文件 `/etc/chrony.conf` 核心参数详解

| **配置指令**        | **示例 / 参数**                    | **含义与作用**                                                                              |
| --------------- | ------------------------------ | -------------------------------------------------------------------------------------- |
| **`server`**    | `server 172.25.250.254 iburst` | 指定**单个**上游 NTP 服务器的 IP 或域名。                                                            |
| **`pool`**      | `pool 172.25.250.254 iburst`   | 指定**一组集群** NTP 服务器（域名解析到多个IP 时使用）。                                                     |
| **`iburst`**    | _（修饰符）_                       | **Initial Burst（初始突发）**：当 `chronyd` 启动或与服务器失去连接重新连上时，快速连续发送4~8 个数据包，以在**几秒钟内完成初始时间同步**。 |
| **`driftfile`** | `/var/lib/chrony/drift`        | 指定记录系统时钟偏差速率的文件路径。                                                                     |
| **`makestep`**  | `makestep 1.0 3`               | 如果系统时间误差大于 1 秒，前3 次更新时允许通过“一步跨越”的方式强制跳变修正，而非平滑微调。                                      |
| **`rtcsync`**   | `rtcsync`                      | 启用硬件实时时钟（RTC/CMOS）的定期同步。                                                               |
| **`allow`**     | `allow 192.168.1.0/24`         | （作为服务端时使用）允许特定网段的客户端向本机器同步时间。                                                          |
### 解答：
```bash
# 修改 chrony 服务配置文件，并添加以下服务器配置
vim /etc/chrony.conf
server 172.25.250.254 iburst
# 添加开机自启，重启服务
systemctl enable chronyd --now
systemctl restart chronyd
# 测试
chronyc sources
# 检查输出中题目指定的服务器是否标为 ^*；^? 表示尚未选中或不可达
```
## 8、配置autofs
### 题目：
```text
配置 autofs，按照以下要求自动挂载远程用户的家目录，要求如下: 
1. NFS 服务器172.25.254.254 的共享目录是/rhome,此文件系统包含为用户testuser 预配置的家目录
2. testuser 的家目录是172.25.254.254:/rhome/testuser 
3. testuser 的家目录应该自动挂载到rhome 下的/rhome/testuser 家目录允许用户写入
4. testuser 的密码是 redhat
```
### `autofs`
### 主映射文件`/etc/auto.master`
主映射文件用于告诉`autofs` 守护进程：*监控哪个父目录，以及去哪里查找具体的子挂载规则*。
```bash
/rhome  /etc/auto.user
```
- **`/rhome`**：**基础挂载点（父目录）**。当用户进入或访问`/rhome/` 下的子目录时，`autofs` 会被触发。（注：如果本地不存在`/rhome` 目录，`autofs` 启动时会自动创建它）。- **`/etc/auto.user`**：*次要映射文件（子配置文件）* 的绝对路径（文件名可以自定义，如 `auto.user` 或`auto.misc`）。
#### 次要映射文件 `/etc/auto.user`
次要映射文件用于定义 **具体子目录的挂载参数和远程设备路径**。
`触发子目录名 -挂载选项 远程 NFS 服务器与路径`

| **字段**     | **本题配置**                         | **含义与作用**                                                                                         |
| ---------- | -------------------------------- | ------------------------------------------------------------------------------------------------- |
| **触发子目录名** | `testuser`                       | 当用户访问`/rhome/testuser` 时触发挂载。                                                                     |
| **挂载选项**   | `-rw,sync`                       | **必须以连字符 `-` 开头**。<br><br>  <br><br>•`rw`：读写权限（题目要求“允许用户写入”）。<br><br>  <br><br>•`sync`：同步写入磁盘与网络。 |
| **远程路径**   | `172.25.254.254:/rhome/testuser` | NFS 服务器地址及其导出的绝对路径。                                                                               |
题目有时会要求：**“使所有远程用户的家目录都能在 `/rhome` 下自动挂载”**。此时如果使用明确的单用户名（如`testuser`），遇到其他用户时就失效了。使用*通配符写法*可以一步到位：
```bash
*  -rw,sync  172.25.254.254:/rhome/&
```
- **`*`（星号）**：代表匹配任意输入的子目录名（例如输入`cd /rhome/user1`，`*` 匹配为`user1`）。- **`&`（和号）**：代表引用`*` 所匹配到的字符串（即自动替换为 `user1`）。

### 解答：
```bash
## 安装autofs
yum install autofs -y
## 创建目录
mkdir /rhome
## 修改主要映射文件
vim /etc/auto.master 
/rhome /etc/auto.user
## 修改次要映射文件
vim /etc/auto.user
testuser -rw,sync 172.25.254.254:/rhome/testuser
## 设置开启自启并重启服务
systemctl enable autofs --now
systemctl restart autofs

df -h /rhome/testuser
```
## 9、配置用户账户
### 题目：
```text
配置用户账户 alex，用户的 ID 为1234，此用户的密码应当为 redhat
```
### `useradd`常见参数

| **需求**                 | **对应参数** | **命令示例**                         |
| ---------------------- | -------- | -------------------------------- |
| 指定**主组**               | **`-g`** | `useradd -g devgroup alex`       |
| 指定**附加组**（属于多个组）       | **`-G`** | `useradd -G sysmgrs,admins alex` |
| 指定**登录 Shell**（不可登录用户） | **`-s`** | `useradd -s /sbin/nologin alex`  |
| 指定**家目录路径**            | **`-d`** | `useradd -d /custom/alex alex`   |
| 指定**用户id**             | -u       | useradd -u 1234 alex             |
**修改已有用户**应使用`usermod`，例如usermod -aG group example为添加附属组；`-G`为直接覆盖原本的所有附加组。
### 解答：
```bash
# 创建用户alex，并设置ID为1234
useradd -u 1234 alex
# 修改密码为redhat
echo "redhat" | passwd --stdin alex

# 检查id alex
```
## 10、查找文件
### 题目：
```text
查找属于 harry 用户的所有文件，并拷贝到/root/findfiles 目录
```
### `find`常用参数：
| **筛选类型** | **参数**       | **示例**                       | **含义说明**                              |
| -------- | ------------ | ---------------------------- | ------------------------------------- |
| 按**类型**  | **`-type`**  | `-type f`（文件）/ `-type d`（目录） | 极力推荐显式加上 `-type f`，避免把目录结构打乱。         |
| 按**组**   | **`-group`** | `-group sysmgrs`             | 查找属于指定组的文件。                           |
| 按**大小**  | **`-size`**  | `-size +10M -size -50M`      | 查找大于 10MB 且小于50MB 的文件（`+` 大于，`-` 小于）。 |
| 按**权限**  | **`-perm`**  | `-perm -4000` / `-perm 0777` | 查找包含 SUID 权限（`-4000`）或特定权限的文件。        |
| 按**用户**  | -user        | -user harry                  | 按文件所有者（属主）为 `harry` 进行筛选。             |
### 解答：
```bash
# 创建文件夹mkdir /root/findfiles
# 查找文件
find / -user harry -exec cp -a {} /root/findfiles/ \;
```
**`find / -user harry -exec cp -a {} /root/findfiles/ \;`**
- **`find /`**：从根目录开始全盘搜索。
- **`-user harry`**：按文件所有者（属主）为 `harry` 进行筛选。
- **`-exec ... \;`**：对找到的每一个结果执行后续的 Shell 命令。    
- **`{}`**：占位符，代表`find` 找到的每一个文件/目录的绝对路径。    
- **`\;`**：`-exec` 动作的结束标志（反斜杠`\` 用于转义分号 `;`，防止被 Bash 解析）。
- **`cp -a`**：*归档拷贝（Archive）*。相当于 `-dr --preserve=all`，作用是**完全保留原文件的所有者、所属组、权限属性和时间戳**。如果没有加 `-a`，直接用 `cp`，拷贝出来的文件所有者就会全部变成当前执行命令的 `root`。

## 11、查找字符串
### 题目：
```text
查找文件/usr/share/xml/iso-codes/iso_639_3.xml 中包含字符串 ng 的所有行。
并将所有这些行的内容放到文件root/list 中， /root/list 不得包含空行
```
### `grep` 考场高频参数总结表
|**参数**|**英文全称**|**核心功能与应用场景**|**考场常用组合**|
|---|---|---|---|
|**`-i`**|`--ignore-case`|**忽略大小写**（如同时匹配 `NG`、`Ng`、`ng`）|`grep -i "pattern" file`|
|**`-v`**|`--invert-match`|**反向匹配**（输出不包含该字符串的所有行）|`grep -v '^#' file`（过滤注释）|
|**`-E`**|`--extended-regexp`|**启用扩展正则**（等价于 `egrep`，支持`\|`、`+` 等）|`grep -E 'cat\|dog' file`|
|**`-r` / `-R`**|`--recursive`|**递归检索目录**（查找某目录下所有文件中的字符串）|`grep -rn "main" /etc/`|
|**`-n`**|`--line-number`|**显示行号**（输出匹配内容及其在文件中的行号）|`grep -n "error" log.txt`|
|**`-w`**|`--word-regexp`|**精准单词匹配**（只匹配独立单词 `ng`，不匹配 `singing`）|`grep -w "ng" file`|
|**`-c`**|`--count`|**统计匹配行数**（不输出具体内容，只打印总行数）|`grep -c "failed" log.txt`|

### 解答：
```bash
# 查找
grep ng /usr/share/xml/iso-codes/iso_639_3.xml > /root/list
# 查看文件
ll /root/list
cat /root/list
```
## 12、创建归档
### 题目：
```text
创建名为/root/backup.tar.gz 的tar 包，用来压缩/usr/local 目录
```
### 归档讲解
**打包 vs 压缩**：在Linux 中，**打包（归档）** 和**压缩** 是两个独立的操作。`tar` 命令原本只负责将多个文件合并成一个`.tar` 归档包；结合对应的压缩选项（如 `-z`、`-j`、`-J`）后，`tar` 才能在打包的同时调用系统压缩工具，实现“打包+ 压缩”一步到位。解题标准命令仅需一行：
```bash
tar czf /root/backup.tar.gz /usr/local
```
### 命令结构拆解：
`tar czf [生成的压缩包绝对路径] [目标目录或文件]`
- **`c`** (Create)：创建新的归档文件。
- **`z`** (gzip)：调用`gzip` 机制进行压缩。
- **`f`** (File)：指定生成的归档文件名。
> **关键细节**：**选项 `f` 必须紧跟输出的文件名**！例如可以写为`tar czf /root/backup.tar.gz ...` 或`tar -c -z -f /root/backup.tar.gz ...`
### `tar` （动作选项）：
任何 `tar` 命令必须且只能包含以下三个主要动作之一：

| **选项**   | **名称**      | **作用**           | **考场场景**                  |
| -------- | ----------- | ---------------- | ------------------------- |
| **`-c`** | `--create`  | **创建**新的归档包      | “创建归档压缩文件……”（本题：          |
| **`-x`** | `--extract` | **解压/解包**归档文件    | “将某个 `.tar.gz` 解压到指定目录……” |
| **`-t`** | `--list`    | **列出**归档包内部的文件列表 | 考场上不解压文件直接检查包内内容          |
### 压缩格式对照表考试时题目要求的后缀名如果发生变化，只需切换对应的压缩参数：

| **压缩格式**  | **文件标准后缀名**       | **tar 关键参数** | **对应算法工具** | **格式特点与记忆口诀**                                           |
| --------- | ----------------- | ------------ | ---------- | ------------------------------------------------------- |
| **gzip**  | `.tar.gz` 或`.tgz` | **`-z`**     | `gzip`     | 速度最快，压缩率中等br><br>  <br><br>**口诀**：小写**`z`** 对应**gzip**  |
| **bzip2** | `.tar.bz2`        | **`-j`**     | `bzip2`    | 压缩率高于gzip<br><br>  <br><br>**口诀**：小写**`j`** 对应**bzip2** |
| **xz**    | `.tar.xz`         | **`-J`**     | `xz`       |                                                         |
### 解答：
```bash
yum install gz* xz* bzip* -y
tar czf /root/backup.tar.gz /usr/local/
```
## 13、创建容器镜像
### 题目：
```text
在servera 上以用户身份 contsvc：
使用题目提供的Containerfile，构建名为watcher 的容器镜像
请勿更改 Containerfile 中的内容
```
### `wget` (网络文件下载器)
`wget` 是Linux 下非交互式的网络文件下载工具，支持HTTP、HTTPS、FTP 协议，极其稳定，非常适合在自动化脚本和命令行环境中使用。
### 本题中的 `wget` 参数拆解
`wget -c http://servera:82/Containerfile`
- **`-c`** (`--continue`)：**断点续传**。如果网络中断或文件只下载了一半，再次执行该命令会从断开处继续下载，而不是重新重新开始，保证了大文件下载的稳定性。

### `wget` 常用参数

|**参数**|**英文全称**|**核心功能与应用场景**|**考场常用示例**|
|---|---|---|---|
|**`-c`**|`--continue`|**断点续传**（下载大文件/网络不稳定时必带）|`wget -c http://...`|
|**`-O`**|`--output-document`|**重命名输出**（将下载的文件保存为指定名称）|`wget -O MyFile.txt http://...`|
|**`-P`**|`--directory-prefix`|**指定保存目录**（下载到特定文件夹下）|`wget -P /tmp/ http://...`|
|**`-q`**|`--quiet`|**静默模式**（关闭终端的下载进度条输出）|`wget -q http://...`|
|**`--no-check-certificate`**|-|**忽略 SSL 证书检查**（针对自签名 HTTPS 站点）|`wget --no-check-certificate https://...`|
### `podman` (下一代无守护进程容器引擎)
`podman` 是Red Hat 主推的容器管理工具，语法与Docker 高度兼容（大部分场景下`alias docker=podman`），其最大特点是 **Daemonless（无后台守护进程）** 和**Rootless（支持普通用户无特权运行）**。

### `podman` 四大核心模块

#### 模块一：镜像仓库认证（Registry Authorization）
- **登录仓库**：
```bash
podman login registry.lab.example.com 
# 登录信息会加密保存在普通用户的 `~/.config/containers/auth.json` 文件中
```
- **登出仓库**：
```bash
podman logout registry.lab.example.com
```

#### 模块二：镜像构建与管理（Image Management）
| **命令**                          | **功能说明**   | **考场易错点/ 细节说明**                          |
| ------------------------------- | ---------- | ---------------------------------------- |
| **`podman build -t watcher .`** | **构建镜像**   | **⚠️ 末尾的句点`.` 绝对不能漏！** 代表“以当前目录作为构建上下文”。 |
| **`podman images`**             | **查看本地镜像** | 验证构建结果，检查`REPOSITORY` 与`TAG`。            |
| **`podman rmi watcher`**        | **删除本地镜像** | 若构建镜像重名或错误，可先删除再重新构建。                    |
| **`podman pull <image_name>`**  | **手动拉取镜像** | 测试仓库连通性时使用。                              |
| **`podman search <keyword>`**   | **搜索镜像**   | 查找私有或公共仓库中的镜像。                           |

#### 模块三：容器运行与管理（Container Operations）
RHCSA 考试中除了构建镜像，还会考查**运行与开机自启容器**：
```bash
## 启动一个后台运行的容器
podman run -d --name myapp -p 8080:80 -v /host/data:/container/data:Z rhel:latest
```
- **关键选项拆解**：    
- **`-d`**：后台运行（Detached）。    
- **`--name`**：指定容器名称。    
- **`-p 8080:80`**：**端口映射**（宿主机 8080 端口映射到容器内 80 端口）。    
- **`-v /host/data:/container/data:Z`**：*挂载数据卷*。    
- **SELinux 极重要参数`:Z`**：在 RHEL 系统中挂载本地目录到容器时，**必须在目录后加上 `:Z` 标签**，Podman 会自动调整该目录的SELinux 上下文（改为`container_file_t`），否则容器内部将因为权限拒绝（Permission Denied）无法读写数据！
#### 模块四：Systemd 容器开机自启管理（Rootless 容器开机自启）
在考场中，如果要求普通用户`contsvc` 运行的容器在**开机后自动启动**：
1. **生成 Systemd 服务文件**：
```bash
mkdir -p ~/.config/systemd/user/
podman generate systemd --name myapp --files --new
mv container-myapp.service ~/.config/systemd/user/
```
2. **启用用户服务**：
```bash
systemctl --user daemon-reload
systemctl --user enable --now container-myapp.service
```
3. **开启持久化会话（即使用户注销服务也不退出）**：
```bash
loginctl enable-linger contsvc
```
### 解答：
```bash
# 先在root下安装podman
yum install podman -y
# 若无contsvc用户则自己创建，密码是redhat
# ssh链接contsvc
ssh contsvc@servera
# 登录题目给的仓库
podman login registry.lab.example.com
# 根据链接获取指定容器文件
wget -c http://servera:82/Containerfile
# 构建镜像
podman build -t watcher .
# 检查podman images
```
## 14、将容器配置为服务
### 题目：
```text
在servera 创建一个无根容器以作为 systemd 服务自动启动，要求如下： 
容器名称为rh134 
容器使用上题创建的watcher 镜像 
容器仅为现有用户 contsvc 作为 systemd 服务运行 
服务名为 container-rh134 
系统重新启动后，容器会自动启动本地目录 /opt/rh124 附加到容器/opt/level1 目录 
本地目录 /opt/rh134 附加到容器/opt/level2 目录 
如果服务工作正常，则放入本地 /opt/rh124 或/opt/rh134 目录的文件都将置入容器的 /opt/level1 和/opt/level2 目录
```
### `podman补充`
#### `podman -v`标志位两个Z的区别：

| **标记选项**      | **英文全称**    | **共享属性**  | **SELinux 上下文修改行为**                                                     | **考场应用场景**                        |
| ------------- | ----------- | --------- | ----------------------------------------------------------------------- | --------------------------------- |
| **`:z`** (小写) | **Shared**  | **多容器共享** | 将宿主机目录的SELinux 标签设置为共享类型（`container_file_t`）。多个容器可以*同时挂载并读写*该目录。        | 当题目要求**多个容器**共同访问同一个宿主机目录时使用。      |
| **`:Z`** (大写) | **Private** | **单容器独享** | 将宿主机目录分配一个**当前容器特有的MCS 标签**（如 `container_file_t:c1,c2`）。**只有当前容器**能访问该目录。 | 当挂载的是当前容器**私有数据**（如数据库文件、独占日志）时使用。 |
#### `podman generate systemd` 核心参数总结
`podman generate systemd` 是将已有 Podman 容器快速转换成 Systemd `.service` 文件的高级工具。

| **参数**        | **全称**    | **作用与深度剖析**                                                                                          | **考场必备性** |
| ------------- | --------- | ---------------------------------------------------------------------------------------------------- | --------- |
| **`--files`** | `--files` | 在当前目录下直接生成 `.service` 文件（文件名格式为`container-<容器名>.service`），而不是仅打印到控制台。                                | **必带**    |
| **`--new`**   | `--new`   | **重要机制**：服务每次启动时**重新创建**容器，服务停止时自动删除容器（`podman run` → `podman rm`）。这样可以保证容器始终基于最新镜像干净运行。 | **推荐**    |
| **`--name`**  | `--name`  | 指定目标容器名称（针对现有的运行容器生成配置）。                                                                             | **必带**    |
#### `loginctl enable-linger` 机制详解
在Linux 系统中，普通用户服务（`systemctl --user`）的生命周期与用户会话深度绑定：
```text
[系统启动] ──> 用户 SSH 登录 ──> 激活 systemd --user ──> 容器服务拉起 
									│ 
								(用户注销/断开) 
									▼ 
								销毁所有用户进程 ──> 容器挂掉！
```
- **开启Linger 后**：
```bash
loginctl enable-linger contsvc
```
系统在引导启动（Boot）时，会**提前为`contsvc` 用户初始化一个独立的 Systemd 用户实例**。无需任何 SSH 登录动作，容器服务即随系统启动而启动，彻底实现真正意义上的后台 Daemon 化。
- **常用检查指令**：    
- 查看指定用户 Linger 状态：`loginctl show-user contsvc | grep Linger` （应输出 `Linger=yes`）    
- 关闭 Linger 功能：`loginctl disable-linger contsvc`
### 解答：
```bash
# 创建题目要求的宿主机目录
mkdir -p /opt/rh124 /opt/rh134
# 将目录所有权赋予普通用户contsvc
chown -R contsvc:contsvc /opt/rh124 /opt/rh134
# 切换至contsvc 用户独立会话
ssh contsvc@172.25.250.10

# 启动容器并挂载两个数据卷
podman run -d --name rh134 -v /opt/rh124:/opt/level1:Z -v /opt/rh134:/opt/level2:Z localhost/watcher:latest
# 验证挂载连通性（向宿主机写入，去容器内查看）
touch /opt/rh124/file1 /opt/rh134/file2
podman exec -it rh134 ls /opt/level{1,2}

# 创建并进入Systemd 用户服务配置路径
mkdir -p ~/.config/systemd/user/
cd ~/.config/systemd/user/
# 自动生成 systemd 服务单元文件
podman generate systemd --files --new --name rh134
# 重新加载用户级systemd 进程并设置开机自启
systemctl --user daemon-reload
systemctl --user enable --now container-rh134
# 开启用户进程持久化保持（允许开机无人登录时自动启动：
loginctl enable-linger

# 验证时重启服务器，并ssh进入contsvc，可以发现容器自动启动reboot
ssh contsvc@172.25.250.10
podman ps
systemctl --user status container-rh134
```

# serverb试题
## 1、重置root 密码
### 题目：
```text
将serverb 主机的密码设置成 redhat
```
### 解答：
1. 开机疯狂按上下键，高亮选中救援内核(包含`rescue`字样)，按键盘 **`e`** 键进入编辑模式。
2. 找到以`linux` 开头的那一行（通常非常长，末尾为`rhgb quiet`）。
3. 将光标移至该行末尾`quiet`后面)，添加一个空格后输入 **`rd.break`**。
4. 按组合键 **`Ctrl + x`** 启动系统。
```bash
# 将真实的根文件系统以读写模式（rw）重新挂载
mount -o remount,rw /sysroot
# 将当前Shell 的根目录环境切换到真实系统的 /sysroot 中
chroot /sysroot
# 非交互式重置 root 密码为redhat
echo redhat | passwd --stdin root
# 创建 SELinux 全盘重新标记打标文件
touch /.autorelabel
# 退出chroot 虚拟环境
exit
# 退出initramfs 应急Shell，触发系统继续正常引导
exit
```
## 2、配置系统的软件仓库
### 题目：
```text
配置系统的软件仓库
YUM 的两个存储库的地址分别是
http://content.example.com/rhel9.0/x86_64/dvd/BaseOS http://content.example.com/rhel9.0/x86_64/dvd/AppStream
```
### 解答：
```bash
# 编辑仓库文件
vim /etc/yum.repos.d/rhel.repo
# 内容填写
[BaseOS] 
name=BaseOS 
baseurl=http://content.example.com/rhel9.0/x86_64/dvd/BaseOS 
gpgcheck=0 
[AppStream] 
name=AppStream baseurl=http://content.example.com/rhel9.0/x86_64/dvd/AppStream 
gpgcheck=0

# 验证
yum list all
```

## 3、调整逻辑卷大小
### 题目：
```text
1. 将名字为 rhel 的逻辑卷的大小调整到200MiB，确保文件系统的内容保持不变 
2. 调整后的逻辑卷的大小范围在180MiB 到220MiB 的范围内都是可以接受的
```

### 容量参数定义解析：绝对大小vs 相对大小
使用 `-L` 参数时，符号的不同代表完全不同的含义：

| **参数写法**          | **含义说明**        | **示例（假设当前LV 为100M：**             | **最终容量**      |
| ----------------- | --------------- | -------------------------------- | ------------- |
| **`-L 200M`**     | 调整为*固定绝对值*      | `lvresize -rL 200M /dev/...`     | **200M**      |
| **`-L +200M`**    | 在原基础上**增量扩容**    | `lvextend -rL +200M /dev/...`    | **300M**      |
| **`-l 100%FREE`** | 使用 VG 中*所有剩余空间* | `lvextend -l +100%FREE /dev/...` | 现有容量 + VG剩余全部 |
### 解答：
```bash
# 查找逻辑卷的全称
lsblk -p
# 调整逻辑卷及文件系统大小；lvextend 不能用于缩容
lvresize --resizefs -L 200M /dev/mapper/rhcsa-rhel
# 检测
df -h
lvs /dev/mapper/rhcsa-rhel
```
## 4、创建交换分区
### 题目：
```text
1. serverb 添加一个756MiB 的额外的交换分区。交换分区应在系统启动时自动挂载。
2. 不要删除或以任何方式改动系统上的任何现有交换分区。
```
### `/etc/fstab` 各字段含义拆解表：
| **字段序号** | **字段值示例**              | **含义说明**                             |
| -------- | ---------------------- | ------------------------------------ |
| **1**    | `UUID=70014b68-...`    | 设备标识（推荐使用UUID，比 `/dev/vdb1` 更稳妥）     |
| **2**    | **`none`** 或**`swap`** | 挂载点（Swap 没有挂载目录，填写`none` 或`swap` 均可） |
| **3**    | **`swap`**             | 文件系统类型（固定填 `swap`）                   |
| **4**    | **`defaults`**         | 挂载参数（默认使用`defaults`）                 |
| **5**    | **`0`**                | dump 备份标志（`0` 表示不备份）                 |
| **6**    | **`0`**                | fsck 检查顺序（`0` 表示启动时不检查文件系统）          |
### 解答：
首先通过 `lsblk -p` 查看磁盘布局，确认`/dev/vdb`（5GiB）为空闲磁盘。
```bash
# 进入 /dev/vdb 磁盘交互分区工具
fdisk /dev/vdb
```
在`fdisk` 交互界面中依次输入：

| **交互输入**     | **操作含义**            | **说明 / 考场细节**                              |
| ------------ | ------------------- | ------------------------------------------ |
| **`n`**      | 新建分区 (New)          | 准备划分新分区                                    |
| **`p`**      | 主分区(Primary)        | MBR/DOS 分区表下的主分区                           |
| **`1`** (默认) | 分区编号                | 选择默认编号 1 即可                                |
| _(直接回车)_     | 起始扇区 (First Sector) | 默认以2048 开始对齐                               |
| **`+756M`**  | **终止 Sector / 大小**  | **核心**：必须加 `+` 号，指定大小为`756M`               |
| **`t`**      | 修改分区类型 (Type)       | 更改默认的`83` (Linux) 标签                       |
| **`82`**     | 填入 Swap 编号          | HEX 码`82` 代表 `Linux swap / Solaris`        |
| **`p`**      | 打印分区表(Print)        | 检查输出中是否有`/dev/vdb1 ... 756M 82 Linux swap` |
| **`w`**      | 保存并退出(Write)        | 将分区表写入磁盘物理表层                               |
```bash
# 格式化Swap 分区
mkswap /dev/vdb1
# 记录UUID值例如UUID=70014b68-9397-4a4c-8215-15c2f6db1537
# 配置 /etc/fstab 实现开机自动挂载
vim /etc/fstab
# 写入内容
UUID=70014b68-9397-4a4c-8215-15c2f6db1537 none swap defaults 0 0
# 按 /etc/fstab 激活交换分区
swapon -a
# 直接激活指定的 swap 设备
swapon /dev/vdb1
# 查看当前活跃的swap 空间列表
swapon -s
# 重启后再查看交换分区
reboot
swapon -s
```
## 5、创建逻辑卷
### 题目：
```text
根据如下要求，创建新的逻辑卷： 
1. 逻辑卷的名字 database, 卷组是datastorage, 大小为60 个PE 
2. datastorage 的PE size 是16MiB 
3. 格式化成 ext4 文件系统。并在系统启动时自动挂载到/mnt/data
```
### Linux存储管理
#### LVM 三层结构与常用命令全家桶
LVM 的基本架构为：**物理卷(PV) → 卷组 (VG) → 逻辑卷(LV)**

| **架构层级**    | **查看状态命令** | **缩写/详细查看** | **常用管理命令**                                     |
| ----------- | ---------- | ----------- | ---------------------------------------------- |
| **物理卷(PV)** | `pvs`      | `pvdisplay` | `pvcreate`（创建）, `pvremove`（删除）                 |
| **卷组 (VG)** | `vgs`      | `vgdisplay` | `vgcreate`（创建）, `vgextend`（扩容）, `vgreduce`（缩小） |
| **逻辑卷(LV)** | `lvs`      | `lvdisplay` | `lvcreate`（创建）, `lvextend`（扩展）, `lvreduce`（缩减） |

#### `lvcreate` 参数对比：`-l` (小写) vs `-L` (大写)

| **参数**        | **对应全称**    | **解释与用途**                                  | **本题正确选项**        |
| ------------- | ----------- | ------------------------------------------ | ----------------- |
| **`-l`** (小写) | `--extents` | 指定逻辑卷占用的 **PE 个数**（如 `-l 60` 表示分配 60 个PE）。 | **✅必须使用 `-l 60`** |
| **`-L`** (大写) | `--size`    | 指定逻辑卷的 **具体物理容量**（如 `-L 960M` 或`-L 1G`）。   | ❌题目要求按PE 数分配      |
#### `vgcreate` 的`-s` 参数
- **默认行为**：如果不带`-s` 参数，LVM 默认创建的卷组的 PE 大小为**4MiB**。
- **题目要求**：指定`datastorage` 的PE 大小为**16MiB**。
- **正确命令**：`vgcreate -s 16M datastorage /dev/vdb2`。
- **检查命令**：可以通过 `vgdisplay datastorage | grep "PE Size"` 验证是否正确显示为`16.00 MiB`。
**创建底层分区前先计算目标容量**：60 个 PE × 16 MiB = 960 MiB。
- **划分分区大小**：在 `fdisk` 分区时，给新建分区的容量**必须大于等于 960MiB**（例如给出`+1G` 或`+1024M`），如果分区的空间小于960MiB，后续创建VG 时将无法容纳 60 个PE。
### 解答：
首先通过 `lsblk -p` 查看磁盘状态，确认 `/dev/vdb` 还有剩余空间。
```bash
fdisk /dev/vdb
```
在`fdisk` 交互界面中依次输入：
- **`n`**：新建分区- **`p`**：主分区（编号选择默认的`2`）- **`First sector`**：直接回车（使用默认起始位置）- **`Last sector`**：输入`+1G`（确保容量大于960MiB）- **`t`** → **`2`**：修改2 号分区的类型
- **`8e`**：指定Hex code 为`8e`（Linux LVM）- **`w`**：保存分区表并退出
```bash
# 创建物理卷
pvcreate /dev/vdb2
# 创建卷组，并使用 -s 参数显式指定 PE 大小为16M
vgcreate -s 16M datastorage /dev/vdb2
# 创建逻辑卷，使用小写 -l 参数按 PE 个数分配 60 个 PE
lvcreate -n database -l 60 datastorage
# 验证逻辑卷创建结果（容量应刚好显示为 960M）lsblk -p
# 将逻辑卷格式化为ext4 文件系统
mkfs.ext4 /dev/mapper/datastorage-database
# 查询逻辑卷的 UUID
blkid /dev/mapper/datastorage-database
# 创建目标挂载目录
mkdir -p /mnt/data
# 配置开机挂载
vim /etc/fstab
# 内容
UUID=fc6db72d-abe6-4f3e-9d7a-dbf0b2a97795 /mnt/data ext4 defaults 0 0
# 必须加载一次 /etc/fstab中的挂载信息，如果有报错请进行修改
mount -a

#重启检测reboot
df -h 

```
## 6、配置系统调优
### 题目：
```text
配置 serverb 使用系统推荐的tuned 配置集
```
### `tuned-adm`与预定义 Profile 扩展
`tuned-adm` 是RHEL 用于动态调整操作系统性能参数（内核参数、磁盘调度、CPU 频率、网络缓冲区等）的核心工具。
### `tuned-adm` 常用命令速查表
| **命令**                         | **作用说明**                 | **考场使用频率**  |
| ------------------------------ | ------------------------ | ----------- |
| **`tuned-adm recommend`**      | **查询当前系统硬件环境推荐的Profile** | **必敲（第一步）** |
| **`tuned-adm profile <name>`** | 将系统配置集切换为指定的 `<name>`    | **必敲（第二步）** |
| **`tuned-adm active`**         | 查看当前正在生效的配置集             | **必敲（验证）**  |
| **`tuned-adm list`**           | 列出系统当前支持的所有预置配置集         | 备用查询        |
| **`tuned-adm off`**            | 关闭 tuned 动态调优功能          | 考场禁用        |

#### 常见预定义Profile 含义剖析

|**配置集名称(Profile)**|**适用场景与调优偏向**|
|---|---|
|**`virtual-guest`**|针对虚拟化Guest 虚拟机（如KVM 客户机）进行调优，优化内存共享与磁盘 I/O。|
|**`throughput-performance`**|针对高吞吐量服务端调优（如Web 服务器、文件服务器），最大化 I/O 和计算能力。|
|**`latency-performance`**|针对低延迟需求调优（如实时交易系统、高并发数据库）。|
|**`powersave`**|节能优先模式，降低CPU 频率以减少功耗。|
|**`balanced`**|性能与功耗平衡模式（桌面系统默认）。|
### 解答：
```bash
# 查询系统当前推荐的配置集名称
tuned-adm recommend
# （假设上一步输出为：virtual-guest）# 将推荐的配置集应用为当前系统配置
tuned-adm profile virtual-guest
# 验证当前生效的配置集是否已正确切换
tuned-adm active
```
# 考试可能会出现的试题
## 1、执行程序
### 题目：
```text
主机上有一个程序hello, 
要求运行该程序，屏幕则显示hello rhcsa
```
### 解答：
```bash
# 先查找命令文件；若题目已提供，不要覆盖
which hello
# 若没有就自己新建
vim /usr/bin/hello
# 写入内容
#!/bin/bash
echo "hello rhcsa"
# 给文件添加执行权限
chmod a+rx /usr/bin/hello
# 执行查看效果
hello
/usr/bin/hello
```
## 2、执行程序
### 题目：
```text
创建一个程序文件名为rhcsa（考试不是，根据考试来） 并将它移动到/usr/local/bin/中
通过运行/usr/local/bin/rhcsa 查找/usr/目录下大于30KB且小于50KB的文件，且具有SETUID权限，保存到/root/test/目录下（根据考试的目录名来写：
```
### `find`命令解析
### `find` 条件筛选参数解析
| **参数选项**    | **语法示例**      | **作用与深度剖析**                                                                                       |
| ----------- | ------------- | ------------------------------------------------------------------------------------------------- |
| **`-size`** | `-size -50k`  | **查找小于 50KB 的文件**。<br>单位说明：`k` (小写) = KiB，`M` (大写) = MiB，`G` (大写) = GiB、br>`-` 表示“小于”，`+` 表示“大于”。 |
| **`-size`** | `-size +30k`  | **查找大于 30KB 的文件**。与上述组合后形成区间（30 KiB，50 KiB）。                                         |
| **`-a`**    | `-a`          | **逻辑与(AND)**。显式连接多个条件。在 `find` 命令中，两个条件间**直接留空格默认就是AND 关系**，因此`-a` 显式写出或省略均可。                     |
| **`-perm`** | `-perm -4000` | **查找包含 SUID 权限的文件**。<br>`4000` 是SUID 的八进制数值。前缀的`-` 表示“文件的权限位中**必须包含**此权限（可以比它多，但不能缺少该位）”。         |

### `-exec` 动作与占位符机制
命令后半部分的`-exec cp -a {} /root/test \;` 结构分解如下：`find [路径] [条件] -exec cp -a {} /root/test \;`
- **`-exec ... \;`**：固定语法结构，表示对`find` 搜索到的每一个文件**逐一执行** `-exec` 后面指定的Shell 命令。
- **`{}`**：*占位符*。代表`find` 每次找到的那个文件的具体路径名。
- **`cp -a`**：`-a`（`--archive`）参数至关重要！它会在复制文件的同时**保留原始文件的权限、所有者、时间戳等元数据**。- **`\;`**：*`-exec` 的结束标记*。斜杠`\` 是为了对分号 `;` 进行转义，防止被当前 Shell 提前解释。
### `perm`匹配逻辑
```text
 -perm 4000  与 -perm -4000  与 -perm /4000 的区别
```

|**参数写法**|**匹配逻辑**|**考场适用场景**|
|---|---|---|
|**`-perm 4000`**|**精确匹配**：权限掩码必须**完全刚好等于** `4000`（即只能有SUID，没有任何`r`、`w`、`x` 权限）。|极少使用|
|**`-perm -4000`**|**掩码包含**：文件的权限中**必须至少包含** SUID 掩码位（不管 `rwx` 权限如何）。|**✅最推荐写法**|
|**`-perm /4000`**|**掩码任意匹配**：文件的权限与给出的掩码位有**任意一位匹配**即符合条件（针对单一掩码`4000` 时与 `-4000` 效果一致）。|可备选|
### 解答：
```bash
# 创建目录
mkdir /root/test
# 编辑脚本文件
vim /usr/bin/rhcsa
# 写入脚本
#!/bin/bash
mkdir -p /root/test
find /usr/ -size -50k -a -size +30k -a -perm -4000 -exec cp -a {} 
/root/test \;

# 给予脚本文件运行权限
chmod a+rx /usr/bin/rhcsa
# 运行查看效果
rhcsa
ll /root/test
```
## 3、系统用户密码的有效期
### 题目：
```text
在系统中新增的用户，要求 25 天后密码自动失效，密码过期前 5 天警告
```
### 用户全局配置
### `/etc/login.defs` 常用参数详解
`/etc/login.defs` 是Linux 系统创建新用户（执行 `useradd`）时的全局配置文件。*它仅对修改配置后新创建的用户生效，不会影响既有的老用户。*

| **参数名称**             | **默认常见值**             | **功能描述与作用**                                           |
| -------------------- | --------------------- | ----------------------------------------------------- |
| **`PASS_MAX_DAYS`**  | `99999`               | **密码最长有效天数**。超过此天数密码强制失效并要求修改。                        |
| **`PASS_MIN_DAYS`**  | `0`                   | **修改密码的最短间隔天数**。`0` 表示可以随时修改密码。                       |
| **`PASS_WARN_AGE`**  | `7`                   | **密码到期提醒天数**。密码过期前 N 天，用户登录时会看到变更警告。                  |
| **`UID_MIN`**        | `1000`                | **普通用户UID 的起始值**（RHEL 7/8/9 默认为1000，较旧系统可能为 500）。     |
| **`UID_MAX`**        | `60000`               | **普通用户UID 的最大值**。                                     |
| **`SYS_UID_MIN`**    | `201`                 | **系统用户（System User）UID 的起始值**。                        |
| **`SYS_UID_MAX`**    | `999`                 | **系统用户（System User）UID 的最大值**。                        |
| **`GID_MIN`**        | `1000`                | **普通组 GID 的起始值**。                                     |
| **`GID_MAX`**        | `60000`               | **普通组 GID 的最大值**。                                     |
| **`CREATE_HOME`**    | `yes`                 | **是否默认创建家目录**。设置为 `yes` 时，`useradd` 会自动创建`/home/用户名`。 |
| **`UMASK`**          | `077`                 | **新建家目录的默认权限掩码**。`077` 表示只有所有者具备完全权限（`700`）。          |
| **`ENCRYPT_METHOD`** | `SHA512` / `YESCRYPT` | **密码哈希加密算法**。指定存储在 `/etc/shadow` 中的加密类型。              |

### `/etc/shadow` 文件字段详解
`/etc/shadow` 储存了系统中所有用户的密码哈希与过期策略，每一行用冒号 **`:`** 分割为**9 个字段**：
> **示例行**：`student:$6$xyz...:19750:0:90:7:14:20080:`

| **序号** | **字段名称**     | **示例对应值**   | **含义与详细解释**                                                                                                            |
| ------ | ------------ | ----------- | ---------------------------------------------------------------------------------------------------------------------- |
| **1**  | **用户名**      | `student`   | 对应 `/etc/passwd` 中的登录账号名称。                                                                                             |
| **2**  | **加密密码**     | `$6$xyz...` | **密码哈希**。<br><br>  <br><br>•`$6$` 代表 SHA-512，`$y$` 代表 Yescrypt：<br><br>  <br><br>•`!!` 或`*` 表示**未设置密码或密码被锁定**，无法从本地登录。 |
| **3**  | **上次修改密码时间** | `19750`     | **上次修改密码的具体日期**。<br><br>  <br><br>计算方式：自 **1970 年1 月1 日**（Unix 纪元）起的累计天数。                                               |
| **4**  | **密码最小生存期**  | `0`         | **两次修改密码的最少间隔天数**（对应`PASS_MIN_DAYS`）。`0` 表示随时可修改。                                                                      |
| **5**  | **密码最大生存期**  | `90`        | **密码的最长有效天数**（对应`PASS_MAX_DAYS`）。使用超过此天数后必须强制改密。                                                                       |
| **6**  | **密码警告天数**   | `7`         | **密码到期前给用户的提示天数**（对应`PASS_WARN_AGE`）。                                                                                  |
| **7**  | **密码禁用/宽限期**  | `14`        | **密码过期后的缓冲期天数**（Inactive）。<br><br>  <br><br>密码过期后仍可登录并修改，但超过该宽限期未改，账号将被强制锁定。                                           |
| **8**  | **账号失效时间**   | `20080`     | **账号彻底停用的绝对日期**。<br><br>  <br><br>同样自1970 年1 月1 日起算。到了此日期后，无论密码是否过期，账号均不可用。                                            |
| **9**  | **保留字段**     | _(空白)_      | **系统保留**，供未来功能扩展使用，目前固定留空。                                                                                             |
### `chage` 核心参数与`/etc/shadow` 字段映射表
`chage`（**Ch**ange **Age**）命令是 Linux 系统中专门用于*查看和修改现有用户密码有效期及账号到期策略*的核心工具。它直接作用并修改`/etc/shadow` 文件中的对应字段。

| **命令选项** | **长参数格式**      | **功能说明**                                     | **参数输入示例**                    | **对应 /etc/shadow 中的位置** |
| -------- | -------------- | -------------------------------------------- | ----------------------------- | ----------------------- |
| **`-l`** | `--list`       | **查看**指定用户的密码时效详细信息（非root 用户也可执行）            | `chage -l student`            | -                       |
| **`-m`** | `--mindays`    | 设置两次修改密码的*最小间隔天数*                            | `chage -m 2 student`          | **第4 字段**               |
| **`-M`** | `--maxdays`    | 设置密码的*最长有效天数*（超过需强制改密）                       | `chage -M 90 student`         | **第5 字段**               |
| **`-W`** | `--warndays`   | 设置密码过期前开始发出警告的**提醒天数**                       | `chage -W 7 student`          | **第6 字段**               |
| **`-I`** | `--inactive`   | 设置密码过期后的**宽限期天数**（超期未改则账号锁定）                  | `chage -I 14 student`         | **第7 字段**               |
| **`-E`** | `--expiredate` | 设置**账号彻底停用的过期日期**（格式`YYYY-MM-DD`，`-1` 表示永久有效） | `chage -E 2026-12-31 student` | **第8 字段**               |
| **`-d`** | `--lastday`    | 修改**上一次修改密码的日期**（设为`0` 可强制下次登录改密）            | `chage -d 0 student`          | **第3 字段**               |

### `chage -l` 输出结果与配置文件映射表
当执行`chage -l student` 时，系统会以易读的方式输出当前账号的时效状态。其每一行输出与 `/etc/shadow` 字段的对应关系如下：

|**chage -l 显示的输出行**|**对应含义**|**绑定的chage 修改参数**|**对应 /etc/shadow 字段**|
|---|---|---|---|
|**`Last password change`**|上一次修改密码的日期|`-d`|**第3 字段**|
|**`Password expires`**|密码下一次将要到期的具体日期|（由 `-d` + `-M` 自动计算）|-|
|**`Password inactive`**|密码过期后的账号宽限关闭日期|`-I`|**第7 字段**|
|**`Account expires`**|账号彻底失效/停用的日期|`-E`|**第8 字段**|
|**`Minimum number of days between password change`**|两次修改密码的最小间隔天数|`-m`|**第4 字段**|
|**`Maximum number of days between password change`**|密码最长有效天数|`-M`|**第5 字段**|
|**`Number of days of warning before password expires`**|密码到期前的警告天数|`-W`|**第6 字段**|
### 解答：
```bash
# 打开全局用户配置文件
vim /etc/login.defs
# 修改如下数值
PASS_MAX_DAYS   25
PASS_MIN_DAYS   0
PASS_WARN_AGE   5
# 添加一个测试用户
useradd haha
# 查看/etc/shadow文件
grep haha /etc/shadow
```
## 4、配置sudo提权
### 题目：
```text
为你的系统配置sudo 提权，具体要求如下 
创建 user1 用户，设置用户密码为 test。
授权 user1 组中的任何用户都可以无密码执行管理员命令
```
### 解答：
```bash
# 创建用户
useradd user1
echo "test" | passwd --stdin user1
# 配置sudoers文件
vim /etc/sudoers
# 参考10行的配置写入user1的配置
%user1    ALL=(ALL)    NOPASSWD:ALL
# 使用wq!保存退出
# 切换到user1
su - user1
# 测试无密码使用sudo命令
sudo ls -a /root
```
