---
title: Linux中的用户管理
date: 2025-11-29 17:40:28
tags:
---

本篇文章描述了Linux中用户管理的基础知识，我将用我学到的知识去梳理相关知识点。

<!-- more -->

# 前言
在进行实验时，具体命令详解优先参考官方手册
```bash
# 更新手册内容
mandb
# 查看命令含义
whatis command
# 查看命令的使用方法
command --help
# 查看命令的手册内容(按q退出)
man command
```

ps:首先自然是老生常谈的概念介绍，略显枯燥但不可或缺

# 一、用户及用户组存在的意义
## 1.用户存在的意义
系统中的资源是有限的，如何合理分配资源，这需要3A机制来完成，接下来将简述所谓的3A机制。
**3A机制**
Linux中的3A机制包括身份认证(Authentication)、授权(Authorization)和审计(Accounting)。
这些机制共同构成了系统中最底层的安全架构。
**身份认证(Authentication)**
身份认证是验证用户身份的过程，确保只有合法用户才能访问系统。常见的认证方式包括用户名/密码、数字证书、生物识别等。
**授权(Authorization)**
授权是在用户通过认证后，根据其身份和角色分配相应的权限，决定用户可以访问哪些资源或执行哪些操作。
**审计(Accounting)**
审计记录并监控用户访问和操作系统的活动，以便在发生安全事件时进行追溯和分析。审计日志是安全审计的重要依据。
## 2.用户组存在的意义 
用户组是一个逻辑容器，并不能单独使用，其错在意义在于对用户进行归类和统一授权。
在系统中有两种组的类型:
初始组：用户建立后自动生成的组，是用户的主组。
附加组：用户需要某些权限所加到的组，为了取得改组拥有的权力。
## 3.用户及用户组在系统中存在的方式
在系统中用户和组都是用ID来表示，我们对这个数字成为ID
|UID区间|类型|说明|
|---|---|---|
|0         |超级用户(root)|唯一的特权用户，拥有系统所有权限。|
|1-999     |系统用户(系统账户)|用于运行系统服务（如 nginx、mysql、sshd），不允许登录（默认 shell 为 /sbin/nologin），无家目录或家目录在 /var/ 下。|
|1000-65535|普通用户(操作用户)|手动创建的用户（如日常登录、开发使用），允许登录，家目录默认在 /home/username 下。|

需要注意的是，UID区间并不是绝对的，对于不同的Linux系统，UID有着不同的区间划分。
如果想要知道当前系统的UID区间定义规则，可在/etc/login.defs文件中查看。
执行以下命令可以查看系统UID(用户ID)和GID(用户组ID)
```bash
# 查看UID区间定义规则
cat /etc/login.defs | grep -E 'UID_MIN|UID_MAX|SYS_UID_MIN|SYS_UID_MAX'
# 查看GID区间定义规则
cat /etc/login.defs | grep -E 'GID_MIN|GID_MAX|SYS_GID_MIN|SYS_GID_MAX'
```
比如我使用的RHEL-9.6中的UID和GID区间定义规则如下
![](https://cdn.jsdelivr.net/gh/Lehuang0212/blog-images/images/20251129185535628.png)

**/etc/passwd,/etc/group文件**
系统中的用户都是以ID的形式存在，但是作为操作者对于数字是不敏感的，所以我们为用户设定了文字别名并把这个别名填写到了/etc/passwd文件中，所以/etc/passwd就是用户信息文件，/etc/group就是组信息文件。
如果用户ID在/etc/passwd文件中有记录，那么系统就认为这个用户是存在的，相反系统认为这个用户不存在。
可以使用vim命令来查看用户信息和组信息
```bash
# 用户信息
vim /etc/passwd
# 组信息
vim /etc/group
```

# 二、用户涉及到的系统配置文件
## 1./etc/passwd
Linux系统用户基本信息的核心配置文件，所有用户均可读取，用于存储用户登录名、UID、家目录等基础信息，每行对应一个用户，字段间用冒号（:）分隔，共7列。
|列序号|字段名称|核心含义|
|---|---|---|
|第1列|用户名(Login Name)|用户登录系统的唯一标识名称|
|第2列|密码占位符(Password)|用户密码，目前弃用所以用x占位|
|第3列|用户ID(User ID)|系统内部识别用户的数字编号(权限控制核心)|
|第4列|用户主组ID(Group ID)|用户默认所属主组的数字编号|
|第5列|用户注释(GECOS)|可选附加信息(无实际功能，仅标识用户)|
|第6列|主目录(Home Directory)|用户登录后默认进入的工作目录|
|第7列|用户默认Shell(Login Shell)|用户登录后执行的命令解释器|

可以用vim查看或编辑/etc/passwd文件内容
```bash
vim /etc/passwd
```
新建立的用户一般位于文件末尾，例如:
![](https://cdn.jsdelivr.net/gh/Lehuang0212/blog-images/images/20251130144222065.png)
## 2./etc/group
Linux系统中存储用户组信息的核心配置文件，每行对应一个用户组，字段间用冒号（:）分隔，共包含4个核心列。
|列序号|字段名称|核心含义|
|---|---|---|
|第1列|组名(Group Name)|用户组的唯一标识名称|
|第2列|密码占位符(Password)|组密码，用x占位|
|第3列|组ID(GID)|系统识别用户组的数字标识(唯一)|

可以用vim查看或编辑/etc/group文件内容
```bash
vim /etc/group
```
新建立的用户组一般位于文件末尾，例如:
![](https://cdn.jsdelivr.net/gh/Lehuang0212/blog-images/images/20251130145225554.png)
## 3./etc/shadow
Linux系统用户密码及账号安全配置的核心影子文件，仅 root 用户可读，用于存储加密后的密码和账号有效期等安全参数，每行对应一个用户，字段间用冒号（:）分隔，共9列。
|列序号|字段名称|核心含义|
|---|---|---|
|第1列|用户名(Login Name)|用户登录系统的唯一标识名称|
|第2列|密码哈希(Password Hash)|用户密码的加密字符，如果存在通常用SHA512加密，如果密码开头有！表示用户被锁定|
|第3列|最后密码修改时间(Last Password Change)|密码最后一次被修改的时间，从1970年开始计算（天数）|
|第4列|密码最短有效期(Minimum Password Age)|修改密码后，至少间隔多少天才能再次修改|
|第5列|密码最长有效期(Maximum Password Age)|密码有效期限，到期后必须修改才能登录|
|第6列|密码过期警告期(Password Expiration Warning)|密码到期前，系统开始提醒用户修改的天数|
|第7列|密码过期宽限期(Password Inactivity Period)|密码过期后，用户仍可登录并修改密码的宽限天数|
|第8列|账号失效时间(Account Expiration Date)|账号绝对失效日期，从1970年起的天数，到期后禁用账号|
|第9列|预留字段|始终为空，为未来系统扩展保留|

可以用vim查看或编辑/etc/shadow文件内容
```bash
vim /etc/shadow
```
新建立的用户一般位于文件末尾，例如:
![](https://cdn.jsdelivr.net/gh/Lehuang0212/blog-images/images/20251130150828667.png)
修改密码时，root用户可以通过直接修改shadow文件来修改用户的密码，需要注意的是，需要用密文形式替代第二行。
```bash
# 生成密文(SH512加密)，这里以密码123为例子
openssl passwd -6 123
# 使用vim编辑/etc/shadow文件
vim /etc/shadow
# 将生成的密文复制，覆盖在第二列并使用wq!强制保存退出即可
```
## 4./etc/skel/.*
用户的环境配置文件（骨文件）
Linux系统的新用户家目录模板目录，其中以 . 开头的隐藏文件（.*）会在通过 useradd新建用户时，自动复制到该用户的家目录（/home/用户名）中，确保所有新用户都拥有统一的基础配置环境（如命令别名、终端提示符、shell 行为等）。
|文件名|文件含义|
|---|---|
|.bash_profile|Bash Shell登录模式的配置文件，仅在首次登录时加载一次。|
|.bashrc|Bash Shell交互模式的核心配置文件，每次打开新终端都会自动加载|
|.bash_history|存储用户在 Bash 终端中执行过的命令历史(默认保留1000条)|
|.bash_logout|用户退出 Bash 登录模式时，自动执行的配置文件|

## 5./home/username
用户的家目录集合
## 6./var/spool/mail/username
用户邮件存放文件
# 三、用户管理
## 1.用户信息查看
```bash
# 查看当前用户
whoami
#查看指定用户ID
id [OPTION]... [USER]...
OPTION:
    -u #查看用户的用户id
    -g #查看用户主组id
    -G #查看用户所有的组的id
    -n #显示名称
```
例如:
![](https://cdn.jsdelivr.net/gh/Lehuang0212/blog-images/images/20251130154216984.png)
## 2.用户切换
```bash
# 切换用户
su 用户名称

# 切换用户及用户环境变量
su - 用户名称

- # 表示切换用户环境
# 如果从root用户切换到其他用户，不需要后者的密码
# 如果从其他用户切换root用户，需要root用户的密码
# 从其他用户切换到另一个用户，需要后者密码
```
需要注意的是，在做用户切换时，使用完用户身份要及时退出，不要在一个shell中反复执行su命令，如果在一个shell中反复执行shell命令会导致环境错乱。
ps:以下频繁切换仅用于举例，不推荐这样运用
例如:
![](https://cdn.jsdelivr.net/gh/Lehuang0212/blog-images/images/20251130155244376.png)
## 3.用户和用户组建立及删除
### (1)用户建立
```bash
# 用户建立
useradd [options] LOGIN
# 常用参数
options:
    -u id username      ##指定UID
    -g id username      ##指定主组GID
    -G id username      ##指定附加组GUID
    -d dir username     ##指定用户家目录
    -M username         ##建立用户时不建立家目录
    -c word username    ##指定用户说明
    -s shell username   ##指定用shell
```
### (2)用户删除
```bash
# 用户删除(此方法无法彻底删除)
userdel username
#用户删除 -r 删除用户的系统配置文件
userdel -r username
# 如果删除用户时没有删除彻底，可以使用rm手动删除残留
rm -fr /home/username /var/spool/mail/username
```
### (3)组建立及删除
```bash
# 组的建立
groupadd [options] GROUPNAME
# 常用参数
options:
    -g GROUPNAME # 指定GID

# 组的删除
groupdel [options] GROUPNAME
```
### (4)用户和组的信息管理
```bash
# 更改组信息
groupmod [options] GROUP
# 常用参数
options:
    -g GID #更改组ID(GID)

# 更改用户信息
usermod [options] LOGIN
#常用参数
options:
    -l      #更改用户名称
    -u      #更改UID
    -g      #更改主组ID
    -G      #更改用户附加组身份
    -aG     #添加用户附加组身份
    -c      #更改用户注释
    -d      #更改家目录指向
    -md     #更改家目录指向同时更改家目录名称
    -s      #更改默认shell
    -L      #冻结用户
    -U      #解锁用户
```
这里不再列举用户和组的建立、删除及信息管理实验，实验时可以用过watch命令监控文件信息变化来验证实验效果。
```bash
watch -n 1 "tail -n 3 /etc/passwd /etc/group;ls -l /home"
# 这个命令可以监控passwd和group文件的信息变化，监控后三行
# ls -l命令监控home目录中的变化
# watch -n 1 表示每秒刷新一次
# 按ctrl+c退出监控
```
监控效果:
![](https://cdn.jsdelivr.net/gh/Lehuang0212/blog-images/images/20251130161105389.png)
### (5)用户认证信息管理
用户认证信息管理中，主要使用passwd和chage命令:
```bash
# passwd的使用
passwd [OPTION...] <accountName>
# 常用参数
OPTION:
    -S      #查看密码状态
    -l      #冻结账号认证
    -u      #解锁账号认证
    -d      #删除密码
    -e      #强制用户下次登录修改密码
    -n      #设置密码最短有效期(期限内不能更改密码)
    -x      #设置密码最长有效期(过期限后必须更改密码)
    -w      #设置密码过期警告期(密码到期前多少天开始警告)
    -i      #设置密码过期宽限期(密码过期多少天仍可登录并更改密码)
# chage的使用
chage [options] LOGIN
options:
    -d      #设置最后密码修改时间
    -m      #设置密码最短有效期(期限内不能更改密码)
    -M      #设置密码最长有效期(过期限后必须更改密码)
    -w      #设置密码过期警告期(密码到期前多少天开始警告)
    -I      #设置密码过期宽限期(密码过期多少天仍可登录并更改密码)
    -E      #设置账号失效时间

# 强制用户下次登录必须修改密码：
passwd -e lehuang
chage -d 0 lehuang
# 设置密码最短有效期为1天
passwd -n 1 lehuang
chage -m 1 lehuang
# 设置密码最长有效期为40天
passwd -x 40 lehuang
chage -M 40 lehuang
# 设置密码过期提前警告时间为2天
passwd -w 2 lehuang
chage -w 1 lehuang
# 设置宽限期为7天
passwd -i 7 lehunag
chage -I 7 lehuang
# 设置账号认证到期时间
chage -E "2026-01-01"
```
这里不再列举用户认证信息管理的实验，实验时可以用过watch命令监控文件信息变化来验证实验效果。
```bash
watch -n 1 "passwd -S lehuang;tail -n 3 /etc/shadow"
```
监控效果:
![](https://cdn.jsdelivr.net/gh/Lehuang0212/blog-images/images/20251130172446642.png)
### (6)权力下放
在系统中普通用户时无法执行系统管理命令的如果需要普通用户执行系统管理动作那么需要root用户来进行授权
```bash
# 普通用户授权方式“sudo”
# 可以使普通用户使用指定用户身份的运行命令

#授权方法：
visudo # 编辑/etc/sudoers并提供语法检测

# 为了代码的规范性我们选择在100行左右添加
# 在lehuang中输入hostname查看当前主机的名称
# lehuang用户在当前主机中以root用户免密执行useradd和userdel
lehuang localhost.localdomain=(root) NOPASSWD:/usr/sbin/useradd,/usr/sbin/userdel 
# 需要注意的是进行操作的普通用户不能在wheel组中
# 不然会被wheel组的规则覆盖
# wheel组是系统管理员组，使普通用户可以使用所有的管理员命令，但必须输入密码
```
测试效果：
![](https://cdn.jsdelivr.net/gh/Lehuang0212/blog-images/images/20251130175055296.png)
![](https://cdn.jsdelivr.net/gh/Lehuang0212/blog-images/images/20251130174929224.png)
# 总结
Linux用户管理的基础内容，写到这里差不多就结束了，作为Linux系统的核心能力之一，用户管理是保障系统安全的基础，打好用户管理的基础，才能为之后的学习打下坚实的地基，虽然命令繁多，但每条命令都有严谨的逻辑，记忆起来并不困难，多练习，多实验，知识也就成为了自己的一部分。今天这篇文章还只是基础起步，随着学习的深入，希望可以写出更多知识梳理的文章。