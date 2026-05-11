---
title: Linux中的文件传输优化
date: 2025-12-07 02:44:34
tags:
---

本篇文章描述了Linux中的文件传输优化的基础知识。

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
在企业中通常我们使用的Linux操作系统不止一个，那么如何在多个Linux操作系统中进行文件传输，这是我们必须掌握的技能，通常在Linux中进行文件传输，我们需要借助网络和一个服务sshd，这个服务通常在Linux系统中是开机启动并且在火墙端接口中放行的。
```bash
# 查看sshd状态
systemctl status sshd
# sshd设定开启的方法
systemctl enable --now  sshd
# 关闭系统防火墙
systemctl disable --now firewalld
```
sshd状态(正常)开启示例:
![](https://cdn.jsdelivr.net/gh/Lehuang0212/blog-images/images/20251207025844616.png)

# 一、系统中的文件传输方法
## 1.scp
scp（Secure Copy）是一个在 Linux、Unix 以及类 Unix 系统中广泛使用的命令，用于在本地主机和远程主机之间安全地复制文件和目录。它基于 SSH（Secure Shell）协议，因此在传输过程中会对数据进行加密，确保数据的安全性。
```bash
# 常用方法
scp [-r] source ... target
scp [-r] <本地文件> <远程主机用户@远程主机ip:远程主机目录的绝对路径>
# 每次传输文件时，都需要输入远程主机用户的密码

EXAMPLE:
scp file root@192.168.78.129:/root/Desktop
scp -r dir root@192.168.78.129:/root/Desktop
# 参数-r表示复制目录
# 参数-q传输文件时不显示进度

# 复制远程文件到本地
scp [-r] <远程主机用户@远程主机ip:远程主机目录的绝对路径> <本地文件>
EXAMPLE:
scp root@192.168.78.129:/root/Desktop/Test /root/Desktop
```
scp传输文件示例(这里开启了进度显示，方便观察):
![](https://cdn.jsdelivr.net/gh/Lehuang0212/blog-images/images/20251207211218913.png)

## 2.rsync
rsync 是一个强大且高效的文件同步和传输工具，既可以通过 SSH 协议进行安全传输，也能使用rsync 守护进程（rsync daemon）进行传输，不仅能实现本地和远程主机间的文件传输，还具备强大的增量传输功能。它会比较源文件和目标文件的差异，仅传输发生变化的部分，从而节省大量的传输时间和带宽。同时，它也支持本地文件系统内的文件同步。
```bash
rsync [OPTION]... SRC [SRC]... DEST
# 常用方法
# 1.复制本地文件到远程主机
rsync [-rlptogD] <本地文件> <远程主机用户@远程主机ip:远程主机目录的绝对路径>
# 2.复制远程文件到本机
rsync [-rlptogD] <远程主机用户@远程主机ip:远程主机目录的绝对路径> <本地文件>

# 常用参数
OPTION:
    -r ##复制目录
    -l ##复制链接
    -p ##复制权限
    -t ##复制时间戳
    -o ##复制拥有者
    -g ##复制拥有组
    -D ##复制设备文件

EXAMPLE:
rsync -r root@192.168.78.129:/mnt /mnt  #将远程mnt目录直接复制到本地mnt目录下
rsync -r root@192.168.78.129:/mnt/ /mnt  #将远程mnt目录中的内容复制到本地mnt目录下
#这里给出/mnt和/mnt/的使用区别：
#ps:观察可发现，只使用-r而没有加上l参数，会自动跳过链接文件 
```

![](https://cdn.jsdelivr.net/gh/Lehuang0212/blog-images/images/20251207212445136.png)
![](https://cdn.jsdelivr.net/gh/Lehuang0212/blog-images/images/20251207212704257.png)

rsync对于批量处理、同步文件往往效率更高，其主要原因是rsync会自己检查对于正在同步的文件，目标是否已经存在，如果已经存在，则不会重复传输，下图以scp和rsync多次传输相同文件做速率对比：
```bash
#设定文件大小，方便实验观察
truncate -s [目标大小] [文件名]
# 将 file 文件大小设置为 500MB，文件不存在则自动创建
truncate -s 500M file
# 对于truncate命令，如果目标大小比原文件大小更小，会直接截断原文件内容造成数据丢失，如果更大则会在原文件末尾自动填充空字符(\0)

dd if=/dev/zero of=[文件名] bs=[块大小] count=[块数量]
# if=/dev/zero：输入文件为 /dev/zero（持续生成空字节的特殊设备）。
# of=[文件名]：输出文件（即要创建 / 修改的文件）。
# bs：单次读写的块大小。
# count：读写的块数量。
# 最终文件大小 = bs × count。
```
![](https://cdn.jsdelivr.net/gh/Lehuang0212/blog-images/images/20251207214704785.png)
# 三、加速文件传输的方法-归档与压缩
## 1.归档
归档是指将多个文件或目录收集整合到一个单独的文件中，这个单独的文件就被称为归档文件。归档的主要目的是方便对大量文件进行管理、备份和传输。
```bash
# tar是Linux 系统中最常用的归档格式之一。它可以将多个文件和目录打包成一个.tar文件，但不进行压缩
tar [OPTION...] [FILE]...
# 常用参数
OPTION:
    c   #创建
    f   #指定文件名称
    x   #解档
    v   #显示过程
    t   #查看
    r   #向归档文件中添加文件
    --get       #解档指定文件
    --delete    #删除指定文件
    -C          #指定解档路径
EXAMPLE:
tar cf etc.tar /etc/            #把etc目录中的内容打包到etc.tar文件
tar tf etc.tar                  #查看etc.tar文件
tar rf etc.tar file             #往etc.tar文件中添加file文件
tar xf etc.tar                  #解档etc.tar文件
tar f etc.tar --get file        #只把file文件从etc.tar文件中解档出来
tar f etc.tar --delete file     #把etc.tar文件中的file文件删除
tar xf etc.tar -C /mnt          #把etc.tar文件解档到/mnt目录中
```
## 2.压缩
压缩文件可以减少存储空间的占用，提升数据传输的效率
Linux 系统默认支持多种压缩格式，以下是一些常见的默认支持格式：
**gz 格式**：以.gz为扩展名，是 Linux 系统中广泛使用的压缩格式。它使用gzip算法进行压缩，具有较快的压缩和解压缩速度，适用于大多数日常文件的压缩。例如，许多软件包在发布时会采用.gz压缩格式。
**bz2 格式**：文件扩展名为.bz2。这种格式采用bzip2算法，压缩率相对较高，但压缩和解压缩速度比gz格式慢一些。在需要更高压缩比的情况下，如对大型数据文件或备份文件进行压缩时，常采用.bz2格式。
**xz 格式**：以.xz为扩展名，使用xz算法进行压缩。它具有极高的压缩率，能最大程度地减小文件体积，但压缩和解压缩过程相对较慢，通常用于对空间要求极为苛刻的场景，如长期存储的大量数据归档。
**zip 格式**：虽然zip格式在 Windows 系统中更为常见，但 Linux 系统也默认支持。它可以将多个文件和目录压缩到一个文件中，并且支持跨平台使用，方便在不同操作系统之间传输文件。此外，zip格式还支持加密功能，可以对压缩文件中的内容进行加密保护。
```bash
# 压缩命令的使用
command [OPTOPN...] target source # 压缩格式 [参数] 压缩之后的文件名 目标文件名
# 解压缩命令的使用
uncommand [OPTOPN...] target # un+压缩格式 [参数] 目标文件名

# 归档压缩可以一步完成
tar -z  #gzip压缩格式
    -j  #bzip2压缩格式
    -J  #xz压缩格式
```