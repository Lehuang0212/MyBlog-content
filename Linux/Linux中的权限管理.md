---
title: Linux中的权限管理
date: 2025-11-30 21:18:46
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

ps:依旧概念起手

# 一、权限的概念
在 Linux 系统的架构里，权限是构建安全堡垒的基石，精准界定了不同用户对文件与目录的操作边界，对系统安全的维护以及数据完整性的保障起着决定性作用。Linux 将权限归纳为读（r）、写（w）和执行（x）这三种基础类别，它们各司其职，共同守护系统资源。
1.**读权限（r）**：对于文件，拥有读权限的用户宛如拥有了一把打开知识宝库的钥匙，能够毫无阻碍地查看文件的具体内容，无论是文本文件中的文字信息，还是配置文件里的参数设置，都能一览无余。切换到目录场景下，读权限就如同赋予了用户进入一间仓库并查看库存清单的权力，用户凭借它可以运用ls命令，清晰罗列目录中的所有文件与子目录，对目录结构与内容分布做到心中有数。
2.**写权限（w）**：当用户对文件持有写权限时，便如同手握一支神奇的笔，能够自由修改文件内容，无论是增添新的文字段落，还是修正既有数据，都能随心操作。值得注意的是，若要删除文件，除了对文件本身有写权限，还需对文件所在目录具备写权限。把目光转向目录，写权限如同给予用户在一片空地上建造房屋的许可，用户可在目录中创建全新的文件与子目录，还能删除目录里已有的文件与子目录，自由规划目录内的资源布局。
3.**执行权限（x）**：若文件属于可执行文件，比如常见的脚本文件或二进制程序，执行权限就像点燃引擎的钥匙，用户得以运行该文件，让程序中的指令依序执行，实现各种功能。在目录方面，执行权限则如同打开一扇门的钥匙，允许用户运用cd命令顺利进入该目录，深入探索目录内部的资源。

# 二、权限的查看及获取
## 1.权限的查看
```bash
# 查看文件权限
ls -l 文件
# 查看目录权限
ls -ld 目录
```
以/etc目录和/etc/passwd为例：
![](https://cdn.jsdelivr.net/gh/Lehuang0212/blog-images/images/20251130213541901.png)
## 2.文件权限的种类
文件的属性被叫做文件的元数据(meta data)，一种元数据用1个byte来记录内容
```bash
#文件权限信息
- | rw-r--r-- | . | 1 | root | root | 1997 | Nov 30 14:48 | /etc/passwd
[1]   [2]      [3] [4]  [5]    [6]    [7]        [8]            [9]

#目录权限信息
d | rwxr-xr-x | . | 136 | root | root | 8192 | Nov 30 21:14 | /etc/
[1]    [2]     [3]  [4]    [5]   [6]    [7]      [8]           [9]
```
### [1]文件类型
|符号|文件类型|
|---|---|
|-|普通文件|
|d|目录|
|l|软连接|
|b|快设备|
|c|字符设备|
|s|socket套接字|
|p|管道(|)|
### [2]文件权限说明
```bash
#用户权限
 rw- | r-- | r--
 [u]   [g]   [o]

# u:User
# g:Group
# o:Other
```
### [3]文件安全上下文标记
```bash
# 系统的selinux开启，那么此位会出现“.”
```
ps:下文仅补充，初学只需了解，后面再深入
**SELinux（Security-Enhanced Linux）**是 Linux 内核级的安全增强模块，本质是一套「强制访问控制（MAC）系统」。它的核心目标是：即使系统被突破（比如 root 权限泄露），也能限制攻击者的破坏范围。
**文件安全上下文（Security Context）**是 SELinux 给每个文件（或目录、进程、端口）附加的「身份标签」，相当于文件的 “SELinux 身份证”。SELinux 判定访问权限时，不依赖传统的 rwx 权限，而是先检查这个 “标签” 是否符合策略规则。
### [4]文件副本标记或目录中子目录标记
```bash
# 对于文件：文件内容被系统记录的次数（硬链接次数）
# 对于目录： 目录中子目录的次数
```
### [5-6]文件的归属
```bash
# 5为文件的拥有者
# 6为文件的拥有组
```
### [7]文件容量统计
```bash
# 对于文件：文件内容大小
# 对于目录：目录中子文件的元数据大小
```
### [8]文件时间戳
```bash
# 文件内容被修改的时间
```
### [9]文件名称
文件名称中一个英文字符占用一个字节，一个中文字符占用三个字节
## 3.用户对于文件的身份识别及设定
### (1)用户对文件的身份
```bash
u:  # #user 文件的拥有者，ls -l 看到的第五列信息
g:  # group 文件拥有组， ls -l 看到的第六列信息
o:  # other 既不是拥有者也不是拥有组成员的其他用户的通称
```
### (2)权限位
```bash
 rw- | r-- | r--
 [u]   [g]   [o]
```
通过对权限的灵活管理，让系统各用户对文件进行读(r)、写(w)、执行(x)操作时有严格的规则限制，从而使整个系统运行更加协调有序。
### (3)文件用户和用户组管理
```bash
# chown基础操作
chown [OPTION]... [OWNER][:[GROUP]] FILE...
# 此格式主要用于复制其他文件的权限设定
chown [OPTION]... --reference=RFILE FILE...
# chown 常用参数
OPTION:
    -R # 更改目录本身及目录中内容的拥有者或者拥有组
EXAMPLE:
chown username filename #更改文件拥有者
chgrp groupname filename #更改文件拥有组
chown username:groupname file #同时更改文件的拥有者和拥有组
chwon -R username dir #更改目录本身及目录中内容的拥有者

# 用以下命令来监控目标文件的权限变化
watch -n 1 "ls -Rl 目标"
```
下图以chown -R username:groupname filename为例
![](https://cdn.jsdelivr.net/gh/Lehuang0212/blog-images/images/20251201003200677.png)

## 4.设定普通权限的方法
```bash
Usage: chmod [OPTION]... OCTAL-MODE FILE...  #最常使用八进制模式
  or:  chmod [OPTION]... --reference=RFILE FILE...  #复制权限
  or:  chmod <a|u|g|o><+|-|=><r|w|x> file  #符号模式
EXAMPLE:
chmod u-rw TEST
chmod u=rwx TEST
chmod -R u=rwx,g=r--,o=r-- TEST

# 八进制模式
rwx=111=7 
rw-=110=6 
r-x=101=5 
r--=100=r=4
-wx=011=3
-w-=010=w=2
--x=001=x=1
---=000=0
# 例如
chmod 055 TEST
```
## 5.系统默认权限设定
系统本身存在的意义是共享资源，从安全角度讲系统共享的资源越少，开放的权力越小系统安全性越高，既要保证系统安全，又要系统创造价值，于是把应该开放的权力默认开放把不安全的权力默认保留。
### (1)如何保留权利
```bash
# umask 表示系统保留权力
umask #查看保留权力
umask OCTAL-MODE #临时设定系统预留权力

# 永久修改方法
vim /etc/bashrc  #shell系统配置文件

# Set default umask to user
if [$UID -gt 199] && ["`id -gn` = `id -un`"];then
        umask 002 # Normal User umask
else    
        umask 077 # Root User umask
fi

vim /etc/profile ##系统环境配置文件

# Set default umask to user
if [$UID -gt 199] && ["`id -gn` = `id -un`"];then
        umask 002 # Normal User umask
else    
        umask 077 # Root User umask
fi

#source作用时使我们更改的内容立即被系统识别 或者source /etc/profile
source /etc/bashrc
```
下图是修改/etc/bashrc(shell系统配置文件)示例
![](https://cdn.jsdelivr.net/gh/Lehuang0212/blog-images/images/20251201010156728.png)
## 6.系统中的特殊权限
在 Linux 系统中，常规的文件权限包括读（r）、写（w）和执行（x），分别对应数字 4、2 和 1。通过这些权限的组合，我们可以灵活地控制用户对文件和目录的访问。然而，在某些特定场景下，常规权限无法满足需求，这时就需要借助特殊权限来实现更精细的访问控制。
### (1)SUID(Set UID)
当一个设置了 SUID 权限的程序被执行时，内核会将进程的有效用户 ID（Effective User ID）临时设置为文件所有者的用户 ID，而实际用户 ID（Real User ID）保持不变。这样，在程序执行期间，该进程就具有了文件所有者的权限。
```bash
# 设置SUID有两种方法
chmod u+s file  #符号法:“+”表示给予，“-”表示移除
chmod 4原权限 file  #数字法，4xxx(xxx表示普通权限数字)
```
下面展示一下修改前后的效果对比：
执行chmod u+s /bin/cat前
![](https://cdn.jsdelivr.net/gh/Lehuang0212/blog-images/images/20251201170336121.png)
执行chmod u+s /bin/cat后
![](https://cdn.jsdelivr.net/gh/Lehuang0212/blog-images/images/20251201170501434.png)
由于 SUID 权限会提升用户的执行权限，不当使用可能会带来安全风险。例如，如果一个恶意程序被设置了 SUID 权限且可被普通用户执行，那么该程序可能会以文件所有者的权限执行任意恶意操作。因此，在设置 SUID 权限时，应谨慎考虑必要性和安全性。
### (2)SGID(Set GID)
对于可执行文件，当设置了 SGID 权限的程序被执行时，进程的有效组 ID（Effective Group ID）会临时被设置为文件所属组的组 ID。对于目录，设置了 SGID 权限后，在该目录下创建的新文件和子目录将自动继承该目录的组，而不是创建者的默认组。
```bash
# 设置SGID有两种方法
chmod g+s file  #符号法:“+”表示给予，“-”表示移除
chmod 2原权限 file  #数字法，4xxx(xxx表示普通权限数字)
```
同样，SGID 权限的不当使用也可能导致安全问题。例如，如果一个可执行文件设置了 SGID 权限且被恶意利用，可能会使普通用户获得过高的组权限，从而访问到不应访问的资源。对于共享目录设置 SGID 权限时，要确保组内成员的权限管理得当，避免权限滥用。
### (3)Sticky Bit
Sticky Bit 权限通过在目录权限上设置一个特殊标志来实现上述功能。它限制了用户对目录中文件的删除和重命名操作，只有满足特定条件的用户才能执行这些操作。
```bash
# 设置stickyid(粘滞位)有两种方法
chmod o+t dir  #符号法:“+”表示给予，“-”表示移除
chmod 1原权限 dir  #数字法，4xxx(xxx表示普通权限数字)
# 需要注意的是，粘滞位是针对目录的，如果一个目录的stickyid开启
# 则目录中的文件只能被文件所有人删除
```
效果展示：
![](https://cdn.jsdelivr.net/gh/Lehuang0212/blog-images/images/20251201172920494.png)
在设置 Sticky Bit 权限时，要明确其适用场景，确保目录中的文件安全性。同时，要注意与其他权限的协同工作，避免因权限设置不当而影响正常的文件操作。
# 三、ACL权限列表
## 1.ACL简介
ACL（Access Control Lists，访问控制列表）是一种灵活的权限管理机制，用于在 Linux 系统中对文件和目录设置更细致的权限。传统的 Linux 文件权限模型基于所有者（owner）、所属组（group）和其他用户（others）进行权限分配，分别对应读（r）、写（w）、执行（x）权限。然而，这种模型在一些复杂的场景下显得不够灵活。例如，当需要给特定用户或组超出传统权限模型的权限时，ACL 就派上了用场。通过 ACL，可以针对单个用户或组设置独立的读、写、执行权限，从而实现更精细的访问控制。
## 2.ACL权限列表的读取
### (1) ACL权限开启标识
```bash
-rw-r--r-- 1 root root 1997 Nov 30 14:48 /etc/passwd
          ^
# 没有“+”代表acl列表未开启
-rw-r--r--+ 1 root root 1997 Nov 30 14:48 /etc/passwd
          ^
# “+”代表acl列表开启
```
### (2)ACL列表权限读取
```bash
getfacl test
# 显示内容分析
# file: test  文件名称
# owner: root 文件拥有者
# group: root 文件拥有组
user::rw-       #文件拥有者权限 
group::r--      #文件拥有组权限
mask::rw-       #能够赋予特殊用户和特殊用户组的最大权限阀值
other::r--      #其他人的权限

#还有一些特殊的，比如
user:username:rw-       #特殊指定用户权限
group:groupname:rw-     #特殊指定的用户组的权限
```
注意：当文件权限列表开启，不要用ls -l 的方式来读取文件的权限！
## 3.ACL列表的控制
```bash
# 语句格式
setfacl [-bkndRLP] {-m|-M|-x|-X...} file ...
# -m 表示设定，例如：
setfacl -m u:lehuang:rw file
setfacl -m g:lehuang:rw file
setfacl -m o:lehuang:rw file
# -x 表示删除，例如：
setfacl -x u:lehuang file
# -b 表示退出ACL列表
setfacl -b file
```
## 4.ACL权限优先级
拥有者 > 特殊指定用户 > 权限多的组 > 权限少的组 > 其他
## 5.mask阈值
```bash
# mask是能够赋予指定用户权限的最大阀值
# 当设定完毕文件的ACL列表之后用chmod缩小了文件拥有组的权力 mask会发生变化
# 恢复：
setfacl -m m:权限 文件
```
## 6.ACL列表的默认权限
设置默认 ACL。**仅对目录有效**，当在该目录下创建新文件或子目录时，新创建的对象将继承该目录的默认 ACL 规则。
```bash
# 对于目录及目录中已存在的内容生效，但是对新建文件不再生效
setfacl -Rm u:lehuang:rw /mnt/pub
# 对于新建文件生效，对于目录及目录中已存在的内容不生效
setfacl -Rm d:u:lehuang:rw /mnt/pub
```
# 四、attr权限
attr 权限，即文件属性权限，是 Linux 系统中一种对文件或目录赋予额外属性的机制。它能进一步增强对文件访问和操作的控制，提供了比传统 r（读）、w（写）、x（执行）权限更细致的管控维度。这些属性可以限制文件的某些操作，如防止文件被删除、修改，或者使其只能追加写入等。
```bash
#attr权限限制所有用户
 i # 不能作任何更改
 a # 能添加不能删除

lsattr dir|file         ##查看attr权限
chattr +i|+a|-i|-a dir|file ##设定attr权限
```