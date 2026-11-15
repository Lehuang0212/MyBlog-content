---
title: "Nginx 基础运用"
date: "2026-04-08T20:01:25+08:00"
tags:
  - Linux
  - Nginx
description: "记录在 Linux 服务器上安装 Nginx、创建站点配置、启用虚拟主机、检查配置和部署 HTTPS 的基本步骤，附常用命令及配置示例。"
---

# 1. 安装 Nginx
```bash
# 安装
sudo apt update && sudo apt install nginx -y
# 测试
which nginx
# 应该会输出类似/usr/sbin/nginx的内容
```
# 2.创建站点配置文件
```bash
sudo vim /etc/nginx/sites-available/example.com
```
**配置文件的内容示例:**
```nginx
# =============================================
# Nginx 站点配置示例 - example.com
# 适用于静态网站或作为反向代理的基础模板
# =============================================

# 定义一个服务器块（一个虚拟主机）
server {
    # ----- 监听端口 -----
    # 监听 IPv4 地址的 80 端口（HTTP 默认端口）
    listen 80;
    # 监听 IPv6 地址的 80 端口（支持 IPv6 访问）
    listen [::]:80;

    # ----- 域名设置 -----
    # 指定该服务器块处理的域名（支持多个，用空格分隔）
    server_name example.com www.example.com;

    # ----- 文档根目录 -----
    # 设置网站文件的存放路径（存放 HTML、图片、PHP 等）
    root /var/www/example.com/html;

    # ----- 默认首页文件 -----
    # 当请求以 / 结尾时，按顺序查找这些文件作为默认页面
    index index.html index.htm index.nginx-debian.html;

    # ----- 日志配置 -----
    # 访问日志：记录所有客户端请求（格式可自定义，此处使用默认 combined）
    access_log /var/log/nginx/example.com_access.log;
    # 错误日志：记录错误信息（级别从 debug 到 crit，此处为默认 warn）
    error_log /var/log/nginx/example.com_error.log;

    # ----- 根路径处理（核心 location）-----
    location / {
        # try_files 按顺序检查文件是否存在：
        # $uri       - 请求的 URI 对应的文件（如 /about.html）
        # $uri/      - 如果请求以 / 结尾，尝试作为目录处理（并查找 index）
        # =404       - 以上都不存在则返回 404 错误
        try_files $uri $uri/ =404;
    }
}
```
# 3.启用站点配置
```bash
# 在.../sites-enabled/目录下创建同名链接文件example.com指向.../sites-available/example.com
sudo ln -s /etc/nginx/sites-available/example.com /etc/nginx/sites-enabled/
# Nginx 默认会加载 /etc/nginx/sites-enabled/ 目录下所有 .conf 文件或普通文件。通过在此目录创建软链接，Nginx 就能读取到 example.com 的配置。
# 输出示例：
# lrwxrwxrwx 1 root root 34 Apr  8 20:00 example.com -> /etc/nginx/sites-available/example.com
# 或是无输出

# 删除默认站点的软连接(可选)
sudo rm -f /etc/nginx/sites-enabled/default
# 避免冲突：默认站点通常监听 80 端口，且 server_name 为 _（捕获所有未匹配的域名），如果不删除，它会与你的 example.com 竞争处理请求，可能导致你的域名访问失败或出现默认欢迎页。
```
# 4.测试配置语法
```bash
# 测试 Nginx 配置文件的语法是否正确。它不会启动或重载 Nginx，只做静态检查。
sudo nginx -t
# 成功示例
# nginx: configuration file /etc/nginx/nginx.conf test is successful
```
**工作原理：**
1.Nginx 读取主配置文件 /etc/nginx/nginx.conf（其中通常会 include 其他目录如 sites-enabled/*）。
2.解析所有 include 的文件，构建完整的配置树。
3.检查语法错误（如缺少分号、大括号不匹配、指令拼写错误）和语义错误（如 listen 端口冲突、引用的文件或目录不存在）。
4.输出测试结果。
# 5.重载 Nginx 服务
```bash
# 平滑重载 Nginx 服务，在不中断现有连接的情况下应用新的配置。
sudo systemctl reload nginx
# 成功则无输出(静默成功)

# 查看状态
systemctl status nginx
```
# 6.配置HTTPS(SSL证书) --可选
Certbot 客户端 是 Let's Encrypt 证书颁发机构（CA）的官方自动化工具，用于申请、续期、撤销 SSL/TLS 证书，并能自动配置 Web 服务器（如 Nginx、Apache）以启用 HTTPS。
```bash
# 安装Certbot 客户端
sudo apt install certbot python3-certbot-nginx -y

# 为域名 example.com 和 www.example.com 申请 Let's Encrypt 证书，并自动修改 Nginx 配置以启用 HTTPS（包括证书路径配置和 HTTP 到 HTTPS 的重定向）。
sudo certbot --nginx -d example.com -d www.example.com
```
执行后，证书及相关文件保存在：
```text
/etc/letsencrypt/live/example.com/
├── cert.pem         # 服务器证书
├── chain.pem        # 中间证书链
├── fullchain.pem    # cert.pem + chain.pem（Nginx 通常使用这个）
├── privkey.pem      # 私钥（权限 600，仅 root 可读）

```

Nginx 配置中引用的是 `fullchain.pem` 和 `privkey.pem`。
