---
title: "Remote-SSH 的使用"
date: "2026-04-29T20:54:24+08:00"
tags:
  - Linux
  - SSH
  - VS Code
description: "记录通过 VS Code Remote-SSH 连接 Linux 云主机的远程开发流程，并补充服务器内存不足时配置 Swap 的步骤和重启后保持生效的方法。"
---

这篇笔记记录了如何使用Remote-SSH在Linux云端主机进行远程开发

<!-- more -->

# 打通远程开发环境(VS code SSH)

1. **安装插件**：在 VS Code 插件市场搜索并安装 `Remote - SSH`。
2. **配置连接**：点击左下角图标><,选择`Connect to Host...` -> `Configure SSH Hosts`。
- **配置文件填入**：
```text
Host ECS_Name
	HostName your_ecs_ip
	User your_user
	IdentityFile C:\Users\your_user\.ssh\id_ed25519
```
- **注意（如果遇到访问问题）**：
* Windows下的私钥权限需要收紧，才可以正常使用Remote-SSH
1. 在 Windows 文件资源管理器中找到：E:\Key\your_private_key
2. 右键点击文件 -> **属性** -> **安全** 选项卡 -> 点击下方的 **高级**。
3. **禁用继承**：点击“禁用继承”按钮，在弹窗中选择“将已继承的权限转换为此对象的显式权限”。
4. **清理用户**：在用户列表中，**删除**除你本人（你的 Windows 用户名）和 `SYSTEM` 以外的所有条目（特别是要把 `Authenticated Users`、`Users` 之类的全部移除）。
5. **确认权限**：确保你本人的权限是“完全控制”或“读取”。
6. 点击确定，再次尝试 VS Code 连接。
# 配置虚拟内存 (Swap)(可选)
对于小型服务器来说，在编译 Go 项目或运行多个 Docker 容器时，很容易触发 OOM (Out of Memory) 导致进程崩溃。配置 2G 的 Swap 可以显著提升系统稳定性。
**配置方法：**
```bash
# 在根目录下迅速预留一个 2GB 大小的文件
# fallocate不像 `dd` 命令那样通过写入零来创建文件，而是直接在文件系统中分配磁盘块，速度极快。
sudo fallocate -l 2G /swapfile
# 设置权限，只允许 root 用户读写。
sudo chmod 600 /swapfile
# 把这个普通的 2GB 文件“格式化”成交换分区格式。
sudo mkswap /swapfile
# 正式挂载启用
sudo swapon /swapfile
# 将挂载信息写入系统启动表。
# 如果不执行这一步，你的服务器一旦重启，Swap 就会失效。写入 `fstab` 后，每次开机系统都会自动帮你执行 `swapon`。
echo '/swapfile none swap sw 0 0' | sudo tee -a /etc/fstab
```
