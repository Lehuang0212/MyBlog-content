---
title: "RHCE 习题解析"
date: "2026-07-28T18:41:07+08:00"
tags:
  - Linux
  - RHEL
  - RHCE
  - Ansible
description: "围绕 RHCE 练习题整理 Ansible 清单、配置、Playbook、角色、存储、模板、Vault 与定时任务的解题过程，并说明常用模块与验证方法。"
---

# 1、安装并配置 Ansible
## 题目：
```text
在控制节点 workstation.lab.example.com 上安装和配置 Ansible： 
安装所需的软件包 
创建静态清单文件 /home/devops/ansible/inventory，要求如下：
	servera 是 dev 主机组的成员 
	serverb 是 test 和 balancers
	serverc 和 serverd 是 prod 主机组的成员 
	prod 组是 webservers 主机组的成员 
创建名为 /home/devops/ansible/ansible.cfg 的配置文件，要求如下： 
	主机清单文件为 /home/devops/ansible/inventory 
	默认内容集合目录为 /home/devops/ansible/mycollections 
	默认角色目录为 /home/devops/ansible/roles
```

## 核心知识点
### `ansible` vs `ansible-navigator`
两者在考试中的定位与区别如下：

|**比较维度**|**ansible (传统 CLI)**|**ansible-navigator (新一代 TUI/CLI)**|
|---|---|---|
|**运行依赖环境**|直接依赖**宿主机**上安装的 Python 及 Ansible 包|依赖**容器执行环境（Execution Environment, EE）**|
|**主要交互方式**|纯命令行终端输出 (stdout)|支持文本用户界面 (TUI) 交互，也可通过 `-m stdout` 输出到标准终端|
|**考场核心使用场景**|快速 Ad-hoc 连通性测试（如 `ansible all -m ping`）|查看图形化主机清单（`inventory --graph`）、调试与运行 Playbook|
|**常用命令示例**|`ansible all -m ping`<br><br>  <br><br>`ansible-doc <module>`|`ansible-navigator run site.yml -m stdout`<br><br>  <br><br>`ansible-navigator inventory -m stdout --graph`|

### `podman login`
在 RHCE 考试环境中，执行 `podman login <镜像仓库地址>` 是**绝对不能遗漏**的初始化步骤，其背后的核心逻辑如下：
- **执行环境（EE）容器化**：`ansible-navigator` 运行 Playbook 时，默认不会直接在控制节点宿主机上跑 Python 脚本，而是会启动一个预装了特定 Python 版本、Ansible Core 以及各类 Collection 的 **Podman 容器** 来执行任务。
- **私有镜像仓库鉴权**：考场环境中的 EE 容器镜像存放于内部私有镜像仓库（例如 `utility.lab.example.com`）。如果未事先使用 `podman login` 完成身份认证，`ansible-navigator` 在拉取或校验 EE 镜像时会因鉴权失败（`401 Unauthorized`）而无法运行任何自动化任务。
###  主机清单文件（`inventory`）语法与结构
主机清单用于定义 Ansible 管理的目标节点。考试中常用 **INI 格式** 编写，其核心语法规则如下表所示：

|**结构 / 语法类型**|**INI 格式写法示例**|**考点与细节解析**|
|---|---|---|
|**单主机指定**|`servera`|直接填写主机的 FQDN 或已解析的主机名/IP。|
|**基础主机组**|`[dev]`<br><br>  <br><br>`servera`|使用 `[组名]` 声明，下方列出该组包含的主机。|
|**多组归属（重叠）**|`[test]`<br><br>  <br><br>`serverb`<br><br>  <br>  <br><br>`[balancers]`<br><br>  <br><br>`serverb`|**同一台主机可同时属于多个组**（例如 `serverb` 既属于 `test` 也属于 `balancers`）。|
|**组嵌套（父子组）**|`[webservers:children]`<br><br>  <br><br>`prod`|**考场必考**：声明父组时必须加上 **`:children`** 后缀；下方只填写**子组组名**（`prod`），不能直接写主机名。|
|**保留组名**|`all` / `ungrouped`|`all` 代表清单中的所有主机；`ungrouped` 代表没有归属于任何自定义组的主机。|

### 主配置文件（`ansible.cfg`）参数拆解
`ansible.cfg` 控制着 Ansible 的核心运行行为（包括读取哪个清单、以谁的身份连接、如何提权等）。

|**配置节点 (Section)**|**参数项 (Key)**|**考场常用设定值**|**作用与配置逻辑**|
|---|---|---|---|
|**`[defaults]`**|`inventory`|`/home/devops/ansible/inventory`|指定默认的主机清单路径（建议使用绝对路径）。|
||`remote_user`|`devops`|指定 SSH 连接受控节点时使用的远端用户名。|
||`roles_path`|`/home/devops/ansible/roles`|指定 Ansible 查找角色（Role）的默认目录。|
||`collections_path`|`/home/devops/ansible/mycollections`|指定 Ansible 查找集合（Collection）的默认目录。|
|**`[privilege_escalation]`**|`become`|`true`|开启提权功能（连接到受控节点后切换身份）。|
||`become_method`|`sudo`|指定提权方式（Linux 环境下固定为 `sudo`）。|
||`become_user`|`root`|指定提权目标用户（通常提权至最高权限 `root`）。|
||`become_ask_pass`|`false`|关闭提权密码询问（因为受控节点已对 `devops` 配置了 `sudo` 免密）。|
## 解答：
```bash
# 安装 Ansible 核心工具及系统角色包（需 sudo 权限）
sudo dnf install ansible ansible-navigator rhel-system-roles -y
# 登录考试环境提供的镜像仓库（账号密码以考场题目提示为准，通常为 admin/redhat）
podman login utility.lab.example.com
# 创建并进入工作目录
mkdir -p /home/devops/ansible
cd /home/devops/ansible
# 编辑主机清单文件
vim /home/devops/ansible/inventory
# 内容
[dev]
servera

[test]
serverb

[balancers]
serverb

[prod]
serverc
serverd

[webservers:children]
prod
# 编写主配置文件
vim /home/devops/ansible/ansible.cfg
# ansible.cfg示例配置文件
ansible-config init --disabled > /tmp/ansible.cfg
# 写入内容
[defaults]
remote_user=devops
inventory=/home/devops/ansible/inventory
collections_path=/home/devops/ansible/mycollections
roles_path=/home/devops/ansible/roles
[privilege_escalation]
become=true
become_ask_pass=false
# 创建集合目录
mkdir /home/devops/ansible/mycollections
# 创建角色目录
mkdir /home/devops/ansible/roles
# 检测
tree /home/devops/ansible/
# 登录容器
podman login utility.lab.example.com
# 查看主机清单是否正确
ansible-navigator inventory -m stdout --graph
# 测试环境
ansible all -m ping
```
# 2、创建 yum 仓库
## 题目：
```text
作为系统管理员，需在受管节点上安装软件。 
创建 playbook /home/devops/ansible/yum_repo.yml，该 playbook 将在各个受管节点上创建以 下 yum 仓库： 
	仓库 1： 
		仓库的名称为 RH294_BASE 
		描述为 RH294 base software 
		BaseURL 为 http://content.example.com/rhel9.0/x86_64/dvd/BaseOS 
		GPG 签名检查为启用状态 
		GPG 密钥 URL 为http://content.example.com/rhel9.0/x86_64/dvd/RPM-GPG- KEY-red hat-release 
		仓库为启用状态 
	仓库 2： 
		仓库的名称为 RH294_STREAM 
		描述为 RH294 stream software 
		BaseURL 为 http://content.example.com/rhel9.0/x86_64/dvd/AppStream 
		GPG 签名检查为启用状态 
		GPG 密钥 URL 为http://content.example.com/rhel9.0/x86_64/dvd/RPM-GPG-KEY-red hat-release 
		仓库为启用状态
```
## 核心模块：`ansible.builtin.yum_repository`
`yum_repository` 模块用于在受控主机的 `/etc/yum.repos.d/` 目录下自动生成和维护 `.repo` 配置文件。

### 模块常用参数表

| **参数项**           | **数据类型** | **考场常用值/示例**                 | **作用与对应 .repo 字段**                                   |
| ----------------- | -------- | ---------------------------- | ---------------------------------------------------- |
| **`name`**        | String   | `RH294_BASE`                 | **必填**。仓库的 ID，对应 `.repo` 文件中的 `[RH294_BASE]` 节点名。    |
| **`description`** | String   | `RH294 base software`        | 仓库的文字描述，对应 `.repo` 中的 `name = ...` 字段。               |
| **`baseurl`**     | String   | `http://.../BaseOS`          | 软件源的 HTTP/FTP/FILE 路径，对应 `baseurl = ...`。            |
| **`gpgcheck`**    | Boolean  | `yes` / `no`                 | 是否开启 GPG 签名校验，对应 `gpgcheck = 1` 或 `0`。               |
| **`gpgkey`**      | String   | `http://.../RPM-GPG-KEY-...` | GPG 公钥的 URL 地址，对应 `gpgkey = ...`。                    |
| **`enabled`**     | Boolean  | `yes` / `no`                 | 是否启用该软件源，默认为 `yes`。                                  |
| **`file`**        | String   | _(可选，如 `rh294`)_             | 指定生成的 `.repo` 文件名（不含后缀）。若不填，默认将配置写入 `[name].repo` 中。 |
### `ansible-doc` 查阅模块参数
考场上如果记不清模块参数的拼写（如 `baseurl` 还是 `base_url`），**不要硬记**，直接通过命令行自带的帮助文档查询：
```bash
# 查询配置 YUM 源对应的模块名称
ansible-doc -l | grep repo
# 查看 yum_repository 模块的详细说明与使用示例
ansible-doc ansible.builtin.yum_repository
```
在打开的文档界面中：
1. 按 `/` 键输入 `/EXAMPLES` 并回车，直达示例章节。
2. 复制官方提供的标准 YAML 结构进行修改即可。
## 解答：
```bash
# 编辑Playbook文件(yaml格式)
vim yum_repo.yml
```
**配置文件如下：**
```yaml
---
- name: install repo
  hosts: all
  tasks:
    - name: RH294_BASE repository
      ansible.builtin.yum_repository:
        name: RH294_BASE
        description: RH294 base software
        baseurl: http://content.example.com/rhel9.0/x86_64/dvd/BaseOS
        gpgcheck: yes
        gpgkey: http://content.example.com/rhel9.0/x86_64/dvd/RPM-GPG-KEY-redhat-release
        enabled: yes

    - name: RH294_STREAM repository
      ansible.builtin.yum_repository:
        name: RH294_STREAM
        description: RH294 stream software
        baseurl: http://content.example.com/rhel9.0/x86_64/dvd/AppStream
        gpgcheck: yes
        gpgkey: http://content.example.com/rhel9.0/x86_64/dvd/RPM-GPG-KEY-redhat-release
        enabled: yes

```
**验证：**
```bash
#运行playbook验证
ansible-navigator run yum_repo.yml -m stdout
#只要看到playbook中的两个task（任务）在所有主机上执行即可（任务状态是ok或者changed都是可以的）

#所有的主机都必须有RH294_BASE和RH294_STREAM两个仓库
ansible all -m shell -a "yum repolist"
```
# 3、安装 Collections
## 题目：
```text
以 devops 身份
将以下 collections 从 files/3/目录中安装到workstation.lab.example.com 
redhat-rhel_system_roles-1.16.2.tar.gz 
ansible-posix-1.4.0.tar.gz 
community-general-4.3.0.tar.gz 
集合应安装到默认集合目录 /home/devops/ansible/mycollections
```
## Ansible Collection（集合）

### `ansible-galaxy collection`
`ansible-galaxy` 是管理 Ansible Roles 和 Collections 的官方 CLI 工具。
**常用命令与选项速查表：**

|**命令 / 选项**|**功能与说明**|**考场常用示例**|
|---|---|---|
|**`install`**|安装指定的 Collection（支持本地 `.tar.gz` 包或远端 Galaxy 仓库）|`ansible-galaxy collection install xxx.tar.gz`|
|**`list`**|列出当前配置路径下所有已安装的 Collection|`ansible-galaxy collection list`|
|**`-p` / `--collections-path`**|**显式指定安装目标路径**（覆盖 `ansible.cfg` 中的默认值）|`ansible-galaxy collection install xxx.tar.gz -p /path/to/dir`|
|**`--force` / `-f`**|**强制覆盖安装**（若已存在同名集合）|`ansible-galaxy collection install xxx.tar.gz --force`|

### Collection 目录架构规范
Collection 解压安装后遵循严格的目录组织规范：
`collections_path/ansible_collections/<namespace>/<collection_name>/`
- **Namespace (命名空间)**：标识开发者或组织（如 `redhat`、`community`、`ansible`）。
- **Collection Name (集合名称)**：具体的功能集合（如 `rhel_system_roles`、`general`、`posix`）。
- **Playbook 调用方式**：使用全限定集合名称（FQCN）调用模块，例如 `ansible.posix.synchronize` 或 `community.general.ini_file`。
###  极速安装与验证指令
```bash
# 切入工作目录
cd /home/devops/ansible

# 一键离线安装当前目录下所有 Collection 包
ansible-galaxy collection install *.tar.gz --force -p /home/devops/ansible/mycollections

# 验证安装列表
ansible-galaxy collection list
```
## 解答：
```bash
# 将文件拷贝到/home/devops/ansible/目录，考试时需要自己用wget命令从网站下载
cp /home/devops/files/3/* .
# 安装集合
ansible-galaxy collection install ansible-posix-1.4.0.tar.gz --force -p /home/devops/ansible/mycollections
ansible-galaxy collection install community-general-4.3.0.tar.gz --force -p /home/devops/ansible/mycollections
ansible-galaxy collection install redhat-rhel_system_roles-1.16.2.tar.gz --force -p /home/devops/ansible/mycollections
# 也可以使用一条命令安装三个集合
ansible-galaxy collection install *.tar.gz --force -p /home/devops/ansible/mycollections
# 验证(会生成三个目录)
ls /home/devops/ansible/mycollections/ansible_collections/
ansible community redhat
```
# 4、安装软件包
## 题目：
```text
创建名为 packages.yml 的 playbook： 
	将 php 和 mariadb 软件包安装到 dev、 test 和 prod 组中的主机上 
	将 RPM Development Tools 包组安装到 dev 组中的主机上 
	将 dev 组中主机上的所有软件包更新为最新版本
```
## 本题示例查看
```bash
# 查找dnf模块
ansible-doc -l | grep dnf
# 查看示例
ansible-doc dnf
# 输入/EXAMPLE然后回车进行搜索，即可看到相应模块的示例
```
## 解答：
```bash
# 编辑配置文件
vim packages.yml
```
**配置文件如下：**
```yaml
---
- name: install php and mariadb
  hosts: dev,test,prod
  tasks:
    - name: install php and mariadb
      ansible.builtin.dnf:
        name:
          - php
          - mariadb
        state: latest

- name: install development tools and update
  hosts: dev
  tasks:
    - name: install development tools
      ansible.builtin.dnf:
        name: "@Development Tools"
        state: latest

    - name: update
      ansible.builtin.dnf:
        name: '*'
        state: latest
```
**验证：**
```bash
#运行playbook验证
ansible-navigator run packages.yml -m stdout
#只要看到playbook中的task（任务）在对应主机上执行即可（任务状态是ok或者changed都是可以的）
```
# 5、使用角色
## 题目：
```text
根据以下要求创建名为 selinux.yml 的 playbook： 
在所有受管节点上运行，使用 selinux 角色，
将 SELinux 策略为 targeted，
将 SELinux 状态为 enforcing
```
## 核心技术点

### 1. 系统角色 (RHEL System Roles) 机制
- 红帽官方预先编写并经过严格测试的各种系统管理角色（如 SELinux、Network、Timesync、Storage 等）。
- 在 Ansible 2.9 及后续版本中，系统角色已整合到 Collection 中，其 **FQCN（全限定角色名）** 为：`redhat.rhel_system_roles.selinux`
### 2. SELinux 角色核心变量对照
`selinux` 角色提供了多个预定义的控制变量，最常考的两个如下：

|**角色变量名**|**可选值示例**|**作用对应的配置文件位置**|
|---|---|---|
|**`selinux_state`**|`enforcing` / `permissive` / `disabled`|修改 `/etc/selinux/config` 中的 `SELINUX=`|
|**`selinux_policy`**|`targeted` / `mls`|修改 `/etc/selinux/config` 中的 `SELINUXTYPE=`|

## 解答：
```bash
# 编辑配置文件
vim selinux.yml
```
**yml配置文件内容如下：**
```yml
---
- name: selinux
  hosts: all
  vars:
    selinux_policy: targeted     # 策略：targeted / mls
    selinux_state: enforcing     # 状态：enforcing / permissive / disabled
  roles:
    - redhat.rhel_system_roles.selinux
```
```bash
# 验证测试
ansible-navigator run selinux.yml -m stdout
```
# 6、使用Timesync RHEL系统角色
## 题目：
```text
安装 RHEL 系统角色软件包，并创建符合以下条件的playbook， /home/devops/ansible/timesync.yml： 
1. 在所有受管节点上运行 
2. 使用 timesync 角色 
3. 配置该角色，以使用当前有效的 NTP 提供 
4. 配置该角色，以使用时间服务器 classroom.example.com
```
## 模块 vs 角色 vs 角色变量
在编写 Playbook 时，首先需要分清以下三个核心概念：

|**概念名称**|**英文名称**|**本质定位**|**本例中的代表**|**核心作用与运行机制**|
|---|---|---|---|---|
|**模块**|**Module**|最小执行单元|`ansible.builtin.dnf`<br><br>  <br><br>`ansible.builtin.copy`|Ansible 的“原子工具”，负责直接在远端主机上执行具体的单步操作。|
|**角色**|**Role**|自动化配置大礼包|`redhat.rhel_system_roles.timesync`|官方或第三方预先封装好的复杂任务集合，内部包含了完整的逻辑、执行任务与配置模板。|
|**角色变量**|**Role Variable**|角色配置开关/入参|`timesync_ntp_servers`|角色暴露给外部调用的“配置接口”，用来给角色传递具体的运行参数。|
## 解答：
```bash
# 确保已经安装了rhel-system-roles
dnf install -y rhel-system-roles
# 下方 Playbook 直接使用已安装的 redhat.rhel_system_roles.timesync 集合角色
```
**编辑playbook：**
```yml
# timesync.yml
---
- name: timesync
  hosts: all
  vars:
    timesync_ntp_servers:
      - hostname: classroom.example.com
        iburst: yes
  roles:
    - redhat.rhel_system_roles.timesync
```
```bash
# 运行playbook测试
ansible-navigator run -m stdout timesync.yml
```
# 7、使用 Ansible Galaxy 安装角色
## 题目：
```text
创 建 /home/devops/ansible/roles/requirements.yaml 文件用以下载并安装角色 到/home/devops/ansible/roles/ 目录中： 
	角色名:balancer，来自于 files/6/haproxy.tar（考试写的是http://materials.example.com/t ools/haproxy.tar） 
	角色名:phpinfo，来自于 files/6/phpinfo.tar（考试写的是http://materials.example.com/to ols/phpinfo.tar）
```
## `ansible-galaxy`
|**命令片段**|**全称 / 参数**|**本质作用**|**考场与生产解释**|
|---|---|---|---|
|**`ansible-galaxy`**|工具名称|Ansible 官方提供的角色/集合管理 CLI 工具|专门用来搜索、下载、创建和管理 Role 及 Collection。|
|**`install`**|子命令|执行“安装”动作|告诉工具接下来要进行下载与解压安装。|
|**`-r`**|`--role-file`<br><br>  <br><br>_(或 `--requirements-file`)_|**指定依赖清单文件**|告诉工具：“不要让我一个个手动输入角色链接，**请按照这个文件里写的列表去批量下载**。”|
|**`roles/requirements.yaml`**|文件路径|清单文件的相对路径|存放角色 `src`（下载源）和 `name`（重命名）的 YAML 文件。|
|**`-p`**|`--roles-path`|**指定安装目标目录**|告诉工具：“下载解压后的角色文件夹，**必须存放到这个指定的目录下**。”|
|**`roles/`**|目录路径|目标保存目录|即当前工作目录下的 `roles/` 文件夹（对应绝对路径 `/home/devops/ansible/roles/`）。|
## 解答：
```yaml
# /home/devops/ansible/roles/requirements.yaml
# src在考试时应为网址
---
- src: file:///home/devops/ansible/haproxy.tar
  name: balancer
- src: file:///home/devops/ansible/phpinfo.tar
  name: phpinfo
```
```bash
# 安装测试
ansible-galaxy install -r roles/requirements.yaml -p roles/
```
# 8、创建和使用角色
## 题目：
```text
根据下列要求，在 /home/devops/ansible/roles 中创建名为 apache 的角色： 
	安装 httpd 软件包，开机启动并立即运行 
	开启防火墙并允许 web 服务通信 
	模板文件 index.html.j2 用于创建/var/www/html/index.html,含有以下内容
		Welcome to HOSTNAME on IPADDRESS，其中 HOSTNAME 和 IPADDRESS 来自于事实 变量 
创建名为 newrole.yml 的 playbook，并使用此角色，这个 playbook 运行于 webservers 主机组
```
## 解答：
```bash
# 初始化角色目录结构
cd /home/devops/ansible/roles/
ansible-galaxy init apache
# 编辑Jinja2 模板文件
vim /home/devops/ansible/roles/apache/templates/index.html.j2
# 写入
Welcome to {{ ansible_fqdn }} on {{ ansible_default_ipv4.address }}
# 编写角色的主任务逻辑 (tasks/main.yml)
vim apache/tasks/main.yml
```
**主角色逻辑：**
```yml
---
# 1. 安装 HTTPD 软件包
- name: install httpd rpm
  ansible.builtin.dnf:
    name: httpd
    state: present

# 2. 安装 Firewalld 防火墙（部分系统可能默认没装，加上更保险）
- name: install firewalld
  ansible.builtin.dnf:
    name: firewalld
    state: present

# 3. 启动 HTTPD 并设置开机自启
- name: Start service httpd, if not started
  ansible.builtin.service:
    name: httpd
    state: started
    enabled: true

# 4. 启动 防火墙 并设置开机自启
- name: Start service firewalld, if not started
  ansible.builtin.service:
    name: firewalld
    state: started
    enabled: true

# 5. 配置防火墙放行 Web 服务 (http)
- name: firewall rule
  ansible.posix.firewalld:
    service: http
    permanent: yes
    state: enabled
    immediate: yes

# 6. 使用 Jinja2 模板生成 index.html
- name: Create a index.html
  ansible.builtin.template:
    src: index.html.j2
    dest: /var/www/html/index.html
```
**编辑ansible目录下的playbook：**
```yml
# newrole.yml
---
- name: httpd service
  hosts: webservers          
  roles:
    - apache                 
```
# 9、从 Ansible Galaxy 使用角色
## 题目：
```text
创建名为 roles.yml 的 playbook，满足以下要求： 
Playbook 运行在 balancers 主机组上，并使用 balancer 角色，这个角色为 webservers 主机组配 置负载均衡 web 服务器，配置完成后，当访问 http://serverb.lab.example.com 将显示以下输出信息： 
	Welcome to serverc.lab.example.com on 172.25.250.12 
	Welcome to serverd.lab.example.com on 172.25.250.13 (刷新网页后) 
这个 playbook 中另一个 play 使用 phpinfo 角色在 webservers 主机组上，执行完毕后，访问主 机组内每台主机的/hello.php 显示以下信息： 
	Hello World from FQDN 
	FQDN 来自于事实变量 
页面同时显示每台受控主机的 PHP 配置
```
## 解答：
**编辑playbook：**
```yml
# roles.yml
- name: phpinfo
  hosts: webservers
  roles:
    - phpinfo

- name: balancers
  hosts: balancers
  roles:
    - balancer
  tasks:
    - name: permit port 80
      ansible.posix.firewalld:
        service: http
        permanent: yes
        state: enabled
        immediate: yes
```
**测试：**
```bash
curl http://serverb.lab.example.com
Welcome to serverc.lab.example.com on 172.25.250.12
curl http://serverb.lab.example.com
Welcome to serverd.lab.example.com on 172.25.250.13
```
# 10、创建和使用分区
## 题目：
```text
根据下列要求，创建名为 partition.yml 的 playbook： 
playbook 中包含一个 play，该 play 在 balancers 主机组中的主机上运行： 
	在设备 vdb 上创建单个主分区，编号为 1，大小为 1500 MiB， 
	使用 ext4 文件系统格式化分区 
	将文件系统永久挂载到 /newpart 如果无法创建请求的分区大小，应显示错误消息： 
		Could not create partition of that size 
	并且应改为使用大小 800 MiB 
	如果设备 vdb 不存在，应显示错误消息 
		Disk does not exist
```
## 思路：
**整个 Playbook 的设计思路可以分为三个层级：**
```text
       ┌─────────────────────────────────────────────────────┐
       │ 1. 检查磁盘 vdb 是否存在                              │
       │    若不存在 -> 仅输出 "Disk does not exist"           │
       └──────────────────────────┬──────────────────────────┘
                                  │
                                  ▼ (若 vdb 存在)
       ┌─────────────────────────────────────────────────────┐
       │ 2. block 块：尝试创建 1500 MiB 分区                   │
       └──────────────────────────┬──────────────────────────┘
                                  │
                      ┌───────────┴───────────┐
                   成功 │                       │ 失败（磁盘小于 1500 MiB）
                        ▼                       ▼
                        │             ┌─────────────────────────────────┐
                        │             │ 3. rescue 块：                 │
                        │             │  - 输出错误提示                 │
                        │             │  - 降级创建 800 MiB 分区        │
                        │             └─────────────────┬───────────────┘
                        │                               │
                        └───────────────┬───────────────┘
                                        │
                                        ▼
       ┌─────────────────────────────────────────────────────┐
       │ 4. always 块：统一进行收尾                            │
       │    - 格式化为 ext4                                   │
       │    - 创建 /newpart 目录                              │
       │    - 挂载并写入 /etc/fstab                           │
       └─────────────────────────────────────────────────────
```
**4 个核心模块：**

|**场景**|**模块名称**|**关键参数解释**|
|---|---|---|
|**创建分区**|`community.general.parted`|`device: /dev/vdb`<br><br>  <br><br>`number: 1` _(分区号)_<br><br>  <br><br>`part_end: 1500MiB` _(分区结束位置/大小)_<br><br>  <br><br>`state: present`|
|**格式化**|`community.general.filesystem`|`fstype: ext4` _(文件系统类型)_<br><br>  <br><br>`dev: /dev/vdb1` _(注意是第一个分区 vdb1)_|
|**创建目录**|`ansible.builtin.file`|`path: /newpart`<br><br>  <br><br>`state: directory`|
|**持久化挂载**|`ansible.posix.mount`|`path: /newpart` _(挂载点)_<br><br>  <br><br>`src: /dev/vdb1` _(设备)_<br><br>  <br><br>`fstype: ext4`<br><br>  <br><br>`state: mounted` _(自动写入 /etc/fstab 并立即挂载)_|
**查看示例:**
```bash
ansible-doc debug 
ansible-doc parted 
ansible-doc filesystem 
ansible-doc file 
ansible-doc mount
```
## 解答：
**编辑playbook：**
```yml
# partition.yml
---
- name: partition
  hosts: balancers
  tasks:                                           
    - name: check device vdb
      ansible.builtin.debug:
        msg: Disk does not exist
      when: ansible_facts['devices']['vdb'] is not defined

    - name: creat part
      block:
        - name: 1500MiB
          community.general.parted:
            device: /dev/vdb
            number: 1
            state: present
            part_end: 1500MiB
          when: ansible_facts['devices']['vdb'] is defined

      rescue:                                     
        - name: dis err
          ansible.builtin.debug:
            msg: Could not create partition of that size 

        - name: 800MiB
          community.general.parted:
            device: /dev/vdb
            number: 1
            state: present
            part_end: 800MiB
          when: ansible_facts['devices']['vdb'] is defined

      always:
        - name: fstype ext4
          community.general.filesystem:
            fstype: ext4
            dev: /dev/vdb1
          when: ansible_facts['devices']['vdb'] is defined

        - name: create dir
          ansible.builtin.file:
            path: /newpart
            state: directory
          when: ansible_facts['devices']['vdb'] is defined

        - name: Mount device
          ansible.posix.mount:
            path: /newpart
            src: /dev/vdb1
            fstype: ext4
            state: mounted
          when: ansible_facts['devices']['vdb'] is defined
```
**测试：**
```bash
ansible balancers -m shell -a 'df -h /newpart'
```
# 11、创建和使用逻辑卷
## 题目：
```text
创建一个名为/home/devops/ansible/lv.yml 的playbook，它将在所有受管节点上运行以执行下列任务 
1. 创建符合以下要求的逻辑卷： 逻辑卷创建在 research 卷组中 逻辑卷名称为 data 逻辑卷大小为 600MiB 
2. 使用 ext4 文件系统格式化逻辑卷 
3. 如果无法创建请求的逻辑卷大小，应显示错误消息 
   Could not create logical volume of that size，
并且应改为使用大小 400MiB。 
4. 如果卷组research 不存在 ，应显示错误消息 
   Volume group does not exist。 
5. 不要以任何方式挂载逻辑卷
```
## LVM 与文件系统模块详解
- **`community.general.lvol` 模块**：
    - `vg`: 目标卷组名称（如 `research`）。
    - `lv`: 目标逻辑卷名称（如 `data`）。
    - `size`: 逻辑卷大小。在 `lvol` 模块中，直接填写数字（如 `600` 或 `400`）默认单位即为 **MB/MiB**。
- **`community.general.filesystem` 模块**：
    - `fstype`: 文件系统类型，填写 `ext4`。
    - `dev`: 逻辑卷设备在 Linux 中的绝对路径，规则为 `/dev/<VG名称>/<LV名称>`（即 `/dev/research/data`）。
```yml
# 环境准备(考试不需要)
# lvm_pre.yml
---
- name: lvm_pre
  hosts: dev
  tasks:
    - name: create partition for LVM
      community.general.parted:
        device: /dev/vdb
        number: 1
        flags: [ lvm ]
        state: present
        part_start: 1MiB
        part_end: 800MiB

    - name: create research vg
      community.general.lvg:
        vg: research
        pvs: /dev/vdb1

- hosts: prod
  tasks:
    - name: create partition for LVM
      community.general.parted:
        device: /dev/vdb
        number: 1
        flags: [ lvm ]
        state: present
        part_start: 1MiB
        part_end: 600MiB

    - name: create research vg
      community.general.lvg:
        vg: research
        pvs: /dev/vdb1
```
```yml
# lvm.yml
---
- name: lvm
  hosts: all
  tasks:
    - name: check lvm
      ansible.builtin.debug:
        msg: Volume group does not exist
      when: "'research' not in ansible_lvm.vgs"

    - name: create lvm
      block:
        - name: create lvm 600Mib
          community.general.lvol:
            vg: research
            lv: data
            size: 600
          when: "'research' in ansible_lvm.vgs"

      rescue:
        - name: dis err
          ansible.builtin.debug:
            msg: Could not create logical volume of that size
          when: "'research' in ansible_lvm.vgs"

        - name: create lvm 400Mib
          community.general.lvol:
            vg: research
            lv: data
            size: 400
          when: "'research' in ansible_lvm.vgs"

      always:
        - name: format ext4
          community.general.filesystem:
            fstype: ext4
            dev: /dev/research/data
          when: "'research' in ansible_lvm.vgs"
```
# 12、生成主机文件
## 题目：
```text
将 files/10/hosts.j2 模板文件拷贝到/home/devops/ansible 目录中，完善此模板文件，用以生成受控节点的/etc/myhosts文件 
创建名为 hosts.yml 的 playbook 对 dev 主机组使用此模板文件，/etc/myhosts 文件内容最终如下： 
127.0.0.1 localhost localhost.localdomain localhost4 localhost4.localdomain4 ::1 localhost localhost.localdomain localhost6 localhost6.localdomain6 172.25.250.10 servera.lab.example.com servera 
172.25.250.11 serverb.lab.example.com serverb 
172.25.250.12 serverc.lab.example.com serverc 
172.25.250.13 servera.lab.example.com serverd
```
## `.j2` 文件
- **后缀含义**：`.j2` 代表 **Jinja2**，它是 Python 中最流行的一种**动态模板引擎**。
- **作用**：在 Linux 中，配置（如 `/etc/hosts`、`/etc/nginx.conf`）平时都是**静态文本**。但在自动化运维中，服务器的 IP 或主机名是动态变化的。
- **工作方式**：Ansible 结合 Jinja2 模板，可以在模板里写**变量、循环、判断语句**。当 Ansible 执行时，会将实际数据填入模板，生成一份**标准的纯文本配置文件**，再发放到受控主机上。
### Jinja2 的三大核心语法符号
在 `.j2` 文件中，只有三种固定的标记语法：

| **语法符号**        | **名称**       | **作用 / 示例**                                         |
| --------------- | ------------ | --------------------------------------------------- |
| **`{{ ... }}`** | **变量取值/表达式** | 输出具体内容，如 `{{ hostvars[item]['ansible_hostname'] }}` |
| **`{% ... %}`** | **逻辑控制**     | 写循环或条件判断，如 `{% for ... %}`、`{% if ... %}`           |
| **`{# ... #}`** | **注释**       | 仅在 `.j2` 中可见，生成最终文件时会被自动忽略                          |
### `groups.all` ：Ansible 的主机组字典
`groups` 是 Ansible 系统内部维护的一个特殊变量（魔术变量），它的本质是一个 **Python 字典（Dictionary）**。
### 数据结构
当你在 `inventory`（主机清单文件）中定义了主机和组之后，Ansible 会自动生成 `groups` 字典：
假设你的主机清单是这样的：
```toml
[dev]
servera

[test]
serverb
serverc
```
Ansible 在后台自动解析生成的 `groups` 变量结构其实长这样：
```json
{
  "all": ["servera", "serverb", "serverc", "serverd"],
  "dev": ["servera"],
  "test": ["serverb", "serverc"],
  "ungrouped": ["serverd"]
}
```
### `groups.all` 的含义
- **`groups`**：代表整个主机组字典。
- **`all`**：Ansible 的预留组，代表**清单里的所有主机**。
- **`groups.all`**：就是提取 `all` 这个 Key 对应的列表，最终得到：`['servera', 'serverb', 'serverc', 'serverd']`。
### `['ansible_default_ipv4']['address']` ：嵌套字典取值
本质上是 **Python 提取多层嵌套字典（Nested Dictionary）数据的标准语法**。
###  `ansible_default_ipv4`
在 Ansible 运行剧本时（`TASK [Gathering Facts]` 阶段），它会在受控机器上执行系统探测，搜集到大量系统信息（称为 **Facts**）。
其中，`ansible_default_ipv4` 就是搜集到的**默认 IPv4 网卡信息**。它本身是一个包含了多项属性的子字典：
```json
"ansible_default_ipv4": {
    "address": "172.25.250.10",
    "netmask": "255.255.255.0",
    "gateway": "172.25.250.254",
    "interface": "eth0",
    "macaddress": "52:54:00:00:00:0a",
    "type": "ipv4"
}
```
### 两对 `['...']`
**一层层定位到最里面的数据**：
1. 第一层 `['ansible_default_ipv4']`： 从整台机器收集到的庞大 Fact 数据库中，**锁定默认网卡**这个对象。
2. 第二层 `['address']`： 在网卡对象内部，进一步**精确定位到 `address` 属性**，提取出纯文本的 IP 地址字符串（如 `"172.25.250.10"`）。
如果你写 `['ansible_default_ipv4']['gateway']`，拿到的就是网关地址 `172.25.250.254`。
## 解答：
```bash
# 拷贝文件
cp /home/devops/files/10/hosts.j2 .
# 编辑.j2模板文件
# 添加以下内容
{% for item in groups.all %}
{{ hostvars[item]['ansible_default_ipv4']['address'] }} {{ hostvars[item]['ansible_fqdn'] }} {{ hostvars[item]['ansible_hostname'] }}
{% endfor %}
```
**编辑playbook：**
```yml
# hosts.yml
---
- name: update /etc/myhosts on dev
  hosts: all
  tasks:
    - name: generate file
      ansible.builtin.template:
        src: hosts.j2
        dest: /etc/myhosts
      when: inventory_hostname in groups.dev
```
```bash
# 运行playbook
ansible-navigator run hosts.yml -m stdout
# 测试
ansible dev -m shell -a 'cat /etc/myhosts'
```
# 13、修改文件内容
## 题目：
```text
创建名为 issue.yml 的 playbook，要求如下: 
此 playbook 运行到所有受控节点 
此 playbook 覆盖/etc/issue 文件中的内容： 
在 dev 主机组的主机中，内容是： Development 
在 test 主机组的主机中，内容是： Test 
在 prod 主机组的主机中，内容是： Production
```
## 解答：
**编辑playbook：**
```yml
---
- name: update issue
  hosts: all
  tasks:
    - name: update /etc/issue
      ansible.builtin.copy:
        content: Development
        dest: /etc/issue
      when: inventory_hostname in groups.dev

    - name: update /etc/issue
      ansible.builtin.copy:
        content: Test
        dest: /etc/issue
      when: inventory_hostname in groups.test

    - name: update /etc/issue
      ansible.builtin.copy:
        content: Production
        dest: /etc/issue
      when: inventory_hostname in groups.prod
```
# 14、创建 Web 内容目录
## 题目：
```text
按照下方所述，创建名为 webcontent.yml 的 playbook： 
该 playbook 在 dev 主机组中的受管节点上运行 创建目录 /webdev： 
	所有者为 webdev 组 
	具有常规权限： owner=read+write+execute， group=read+write+execute， other=read+execute 
	具有特殊权限： set group ID 
	用软链接将 /var/www/html/webdev 链接到 /webdev 
	创建文件 /webdev/index.html，内容为： Red Hat Ansible 
	在 dev 主机组中主机上浏览此目录（http://servera.lab.example.com/webdev/index.h tml ）将生成以下输出： 
	Red Hat Ansible
```
## 核心模块

### `ansible.builtin.user` （用户管理）
- **场景**：创建或管理系统用户。
- **常用参数**：
    - `name`: 用户名。
    - 保持幂等性：若用户已存在，执行该 Task 不会产生变更（显示 `ok`）。
### `ansible.builtin.file` （文件/目录与权限管理）
- **场景**：创建目录、创建软链接、修改权限及 SELinux 上下文。
- **常用参数**：
    - `path`: 目标路径（创建目录时使用）。
    - `state`: 类型标识。`directory`（目录）、`link`（软链接）、`absent`（删除）等。
    - `mode`: 权限代码（建议字符串形式，如 `'2775'`）。
        - **SGID 特殊权限**：首位 `2` 表示设置 SGID（Set Group ID），在该目录下新建的文件/子目录会自动继承父目录的所属组。
    - `src` / `dest`: 创建软链接时的对应路径：
        - `src`: 真实物理源路径（例如 `/webdev`）。
        - `dest`: 生成的快捷方式/软链接路径（例如 `/var/www/html/webdev`）。
    - `setype`: 指定文件/目录的 **SELinux 安全上下文类型**（例如 `httpd_sys_content_t`），防止 Web 报 403 错误。

### `ansible.builtin.copy` （文件复制与内联生成）
- **场景**：将本地文件拷贝至受控端，或直接生成文本内容。
- **常用参数**：
    - `content`: 直接写入字符串内容（无需准备本地源文件）。
    - `dest`: 生成文件的绝对路径。
    - `setype`: 同样支持在生成文件时赋予 SELinux 标签。
###  `ansible.posix.firewalld` （防火墙策略管理）
- **场景**：配置系统防火墙放行策略。
- **常用参数**：
    - `service`: 预定义的**网络服务名称**（例如 `http`、`https`）。
    - `permanent: yes`: 写入持久化配置，确保重启节点后依然有效。
    - `state: enabled`: 启用规则。
    - `immediate: yes`: 立即应用规则至当前运行中的防火墙，无需重启 `firewalld`。
####  `ansible.builtin.service` （Systemd 服务控制）
- **场景**：管理系统服务的运行状态（启动、停止、重启等）。
- **常用参数**：
    - `name`: 系统**服务进程名**（例如 `httpd`、`sshd`）。
    - `state`: 服务状态，包含 `started`、`stopped`、`restarted`、`reloaded`。
## 解答：
**编辑playbook：**
```yml
---
- name: webcontent
  hosts: dev
  tasks:
    - name: create webdev group
      ansible.builtin.group:
        name: webdev
        state: present

    - name: create webdev file
      ansible.builtin.file:
        path: /webdev
        state: directory
        owner: root
        group: webdev
        mode: '2775'
        setype: httpd_sys_content_t

    - name: create a symbolic link
      ansible.builtin.file:
        src: /webdev
        dest: /var/www/html/webdev
        state: link

    - name: copy using inline content
      ansible.builtin.copy:
        content: Red Hat Ansible
        dest: /webdev/index.html
        setype: httpd_sys_content_t

    - name: permit traffic in default zone for http service
      ansible.posix.firewalld:
        service: http
        permanent: yes
        state: enabled
        immediate: yes

    - name: restart httpd
      ansible.builtin.service:
        name: httpd
        state: restarted
```
**检测：**
```bash
# 运行playbook
ansible-navigator run webcontent.yml -m stdout
# 检测
curl http://servera.lab.example.com/webdev/index.html
# 应输出Red Hat Ansible
```
# 15、生成硬件报告
## 题目：
```text
创建名为 hwreport.yml 的 playbook，在所有受控主机上生成/root/hwreport.txt 文件，内容包含: 
	清单主机名称 
	以 MB 表示的总内存大小 
	BIOS 版本 
	磁盘设备 vda 的大小 
	磁盘设备 vdb 的大小
	输出文件中的每一行含有一个 key=value 对 
	playbook 应当来自 files/13/hwreport.empty 模板文件，并将它保存为 /root/hwreport.txt 
	使用正确的值修改 /root/hwreport.txt 如果硬件项不存在，相关的值应设为 NONE
```
## 解答：
```bash
# 将模板文件复制到目录下
cp /home/devops/files/13/hwreport.empty .
# 查看模板文件
cat hwreport.empty
# Hardware report 
HOST=inventoryhostname 
MEMORY=memory_in_MB 
BIOS=BIOS_version 
DISK_SIZE_VDA=disk_vda_size 
DISK_SIZE_VDB=disk_vdb_size
```
**编辑playbook：**
```yml
---
- name: dis report
  hosts: all
  vars:
    hw_all:
      - hw_name: HOST
        hw_cont: "{{ inventory_hostname | default('NONE', true) }}"
      - hw_name: MEMORY
        hw_cont: "{{ ansible_memtotal_mb | default('NONE', true) }}"
      - hw_name: BIOS
        hw_cont: "{{ ansible_bios_version | default('NONE', true) }}"
      - hw_name: DISK_SIZE_VDA
        hw_cont: "{{ ansible_devices.vda.size | default('NONE', true) }}"
      - hw_name: DISK_SIZE_VDB
        hw_cont: "{{ ansible_devices.vdb.size | default('NONE', true) }}"

  tasks:
    - name: copy template hwreport file
      ansible.builtin.copy:
        src: /home/devops/files/13/hwreport.empty
        dest: /root/hwreport.txt

    - name: replace hardware info in hwreport.txt
      ansible.builtin.lineinfile:
        path: /root/hwreport.txt
        regexp: "^{{ item.hw_name }}="
        line: "{{ item.hw_name }}={{ item.hw_cont }}"
      loop: "{{ hw_all }}"
```
**验证：**
```bash
# 运行playbook
ansible-navigator run hwreport.yml -m stdout
# 测试
ansible all -m shell -a 'cat /root/hwreport.txt'
```
# 16、创建 password vault
## 题目：
```text
创建一个名为 locker.yml 的 ansible vault 文件存储用户密码： 
Vault 文件中含有两个变量，名称如下： 
	pw_developer: Imadev 
	pw_manager: Imamgr 
	加解密密码为 redhat 
	密码存储在 /home/devops/ansible/secret.txt 中
```
## 解答：
```bash
# 创建密码文件secret.txt并写入密码，修改文件权限
echo "redhat" > secret.txt
chmod 600 secret.txt
# 修改ansible配置文件，写入新配置
vim ansible.cfg
# 在defaults下添加
vault_password_file=/home/devops/ansible/secret.txt
# 执行命令创建locker.yml文件，写入题目给出的变量信息
ansible-vault create locker.yml
pw_developer: Imadev 
pw_manager: Imamgr 
# 执行命令查看内容
cat locker.yml # 密文
ansible-vault view locker.yml  # 明文
```
# 17、创建用户帐户
## 题目：
```text
需要创建的用户账户清单来自于 files/15/user_list.yml，请拷贝到/home/devops/ansible/中，配合上一题的 locker.yml，创建名为 users.yml 的 playbook，要求如下： 
	用户的 job description 为 developer 的用户，创建到 dev 和 test 主机组中，用户密码来自于 pw_developer 变量，用户的附属组是 devops 
	用户的 job description 为 manager 的用户，创建到 prod 主机组中，用户密码来自于 pw_manager 变量，用户的附属组是opsmgr 
	用户密码使用 SHA512 hash
```
## 解答：
```bash
# 拷贝用户文件
cp /home/devops/files/15/user_list.yml .
```
**编辑playbook：**
```yml
# users.yml
---
- name: create developer users
  hosts: dev,test
  vars_files:
    - user_list.yml
    - locker.yml
  tasks:
    - name: create devops group
      ansible.builtin.group:
        name: devops
        state: present

    - name: create users
      ansible.builtin.user:
        name: "{{ item.name }}"
        uid: "{{ item.uid }}"
        groups: devops
        append: yes
        password: "{{ pw_developer | password_hash('sha512') }}"
      loop: "{{ users }}"
      when: item.job == "developer"

- name: create manager users
  hosts: prod
  vars_files:
    - user_list.yml
    - locker.yml
  tasks:
    - name: create opsmgr group
      ansible.builtin.group:
        name: opsmgr
        state: present

    - name: create users
      ansible.builtin.user:
        name: "{{ item.name }}"
        uid: "{{ item.uid }}"
        groups: opsmgr
        append: yes
        password: "{{ pw_manager | password_hash('sha512') }}"
      loop: "{{ users }}"
      when: item.job == "manager"
```
**验证：**
```bash
# 运行playbook
ansible-navigator run users.yml -m stdout
# 查看各个主机的用户信息
ansible all -m shell -a "id bob"
ansible all -m shell -a "id sally"
ansible all -m shell -a "id fred"
```
# 18、更新 Ansible 库的密钥
## 题目：
```text
请为 expense.yml 文件修改 vault 密码，要求如下： 
	请将 files/16/expense.yml 文件保存到 /home/devops/ansible/expense.yml 
	当前密码为 ibmibm 
	新密码为 redhatredhat
```
## 解答：
```bash
# 先将ansible.cfg文件中的vault_password_file注释掉，修改密码后再改回来
# 拷贝文件 
cp /home/devops/files/16/expense.yml .
# 修改 vault 密码
ansible-vault rekey expense.yml --ask-vault-pass
# 输出 Rekey successful 就成功了
```
# 19、配置 cron 作业
## 题目：
```text
在所有受管节点上创建名为 /home/devops/ansible/cron.yml 的 Playbook，为 natasha 用户创建 cron 作业： 
	用户 natasha 每隔 2 分钟执行 logger “RH294 in progress”
```
## 解答：
**编辑playbook：**
```yml
---
- name: create cron for natasha
  hosts: all
  tasks:
    - name: create user natasha
      ansible.builtin.user:
        name: natasha

    - name: create cron for natasha
      ansible.builtin.cron:
        name: "natasha cronjob"
        weekday: "*"
        minute: "*/2"
        hour: "*"
        month: "*"
        day: "*"
        user: natasha
        job: '/bin/logger "RH294 in progress"'
```
**验证：**
```bash
# 运行playbook
ansible-navigator run cron.yml -m stdout
# 测试
ansible all -m shell -a "crontab -u natasha -l"
```
