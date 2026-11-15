---
title: "Sing-box 代理部署"
date: "2026-06-23T19:03:14+08:00"
tags:
  - Linux
  - 网络
  - Sing-box
  - 代理
description: "介绍 sing-box 的入站、路由、DNS 与出站配置，并以 VLESS、Reality 和 Cloudflare WARP 为例记录个人代理网络的安装、配置与验证过程。"
---

这是一份基于 `sing-box` 构建个人代理网络的指南，采用 `VLESS + XTLS-Reality` 实现入站流量的伪装与低性能损耗，并结合 `WireGuard` 协议打通 Cloudflare WARP 双栈隧道。
# 一、Sing-box 基础讲解
## 1.定位与优势
`sing-box` 在官方定义中是一个**通用应用层代理平台（Universal Proxy Platform）**。它的出现主要为了解决传统代理工具在协议支持与路由策略上的功能割裂问题。它由 Go 语言开发，其核心优势在于**底层协议栈的完全统一与模块化**。它不仅原生集成了 Xray 的最新加密协议与 Shadowsocks、WireGuard、Trojan 等标准协议，还重写了内存管理和路由调度逻辑。这使得它既能在服务器端处理高并发的隧道封装，也能以极低的资源占用运行在客户端环境中。
[sing-box的项目地址](https://github.com/sagernet/sing-box)
[sing-box的wiki主页](https://sing-box.sagernet.org/)
## 2.流量处理管道
`sing-box` 的核心工作机制是一个标准的数据流处理管道。任何网络数据包经过该内核，均需严格遵循以下四个模块的处理生命周期：**入站 (Inbounds) -> 路由 (Route) -> DNS 解析 -> 出站 (Outbounds)**。
- **Inbounds（入站网关）：**
负责在指定网络接口（Interface）上创建监听套接字（Socket），并处理传入连接的 TLS 解密与协议解包。
 **作用**：验证连接合法性，将不同加密协议（如 VLESS、SOCKS5、HTTP）剥离外层封装，提取出原始的请求负载（Payload）及目标地址（IP/Domain）。
 
 - **DNS（域名解析模块）：**
 内置的异步 DNS 解析引擎，完全接管系统默认的 DNS 请求。
 **作用**：支持基于 UDP/TCP、DoH (DNS over HTTPS)、DoT、DoQ 的解析方式。可配置分流解析策略（Split-horizon DNS），即针对不同后缀的域名向不同的上游 DNS 服务器发起查询，以避免 DNS 污染及缓存投毒（DNS Cache Poisoning）。
 - **Outbounds（出站调度）：**
 执行数据流的重新封装与向外转发。
 **作用**：根据 Route 模块分配的标签，将原始流量套上对应的底层协议帧（如 `WireGuard` UDP 报文或 VLESS TCP 数据流），并发起对目标服务器或下一跳网关的 TCP/UDP 连接。
 - **数据包生命周期过程：**
 当一个数据包（比如你要访问 `www.google.com`）进入你的 VPS 上的 `sing-box` 时，它会经历以下极其严格的加工流水线：
**第一步：** Inbounds（入站剥离）：客户端发来一段由特定协议加密的数据流(比如**目前主流的VLESS传输协议**)，此时内核了解到了目的地址为www.google.com；目的端口为443；协议是TCP。
**第二步**：Route（路由决策与 DNS 触发）：数据包进入 Route 模块。Route 引擎开始拿着你的 `rules`（规则链）自上而下逐条比对，如果规则仅需域名即可匹配（如 `domain_keyword`），则直接分配出站标签；若使用基于目标 IP 的规则，应按当前版本的解析与规则集配置提供可匹配的 IP 信息，再决定出站标签。例如：Outbounds(出站方向) = warp-ipv6-out。
**第三步：** Outbounds（出站封装）：数据包带着标签被送到了 Outbounds 模块。以`warp-ipv6-out`这个出站通道举例，这个方向配置的协议是`WireGuard`，于是数据包按照`WireGuard`协议的标准重新加密，套上新的UDP头部，并发往Cloudflare的边缘节点。
## 3.JSON 配置的声明式范式
`sing-box` 的配置文件耦合性极低使其配置十分灵活，其原因在于它采用了**高度解耦的声明式配置范式（Declarative Configuration）**，通过采用“声明式”和严格的“**Tag（标签）寻址制**”，彻底切断了模块之间的物理绑定。
系统中所有的入站、出站、DNS 服务器，在配置声明时都必须分配一个全局唯一的 `tag`。路由模块在编写分流规则时，绝不包含具体的物理 IP 或协议细节，仅通过 `tag` 进行逻辑映射。
**技术优势**：这种面向对象的配置结构逻辑极其清晰，非常适合采用诸如 `jq` 等自动化流处理器进行编程级修改（Programmable Manipulation）。此外，高度的模块化允许开发者自由构建**代理链（Proxy Chain）**，例如：通过 `selector` 选定节点后，再将流量二次导入一个 `wireguard` 隧道出站。
## 4.核心配置语法
### Log (日志模块)：
```json
"log": {
	"level":"info",
	"timestamp":"true"
}
```
- `level` 分为trace, debug, info, warn, error, fatal。日常使用info,排错使用debug
- `timestamp` 表示是否要启用日志时间戳
### DNS (域名解析)：
```json
"dns": {
	"servers": [
	{
		"tag": "google-dns",
		"type": "udp",
		"server": "8.8.8.8",
		"detour": "direct"
	}
	],
	"final": "google-dns",
	"strategy": "prefer_ipv4"
}
```
- `tag` 为该模块分配一个全局唯一的标识符
- `type` 协议声明
- `server` 指定DNS服务器地址
- `detour` 指定从哪个出站方向发送DNS请求
- `final` 决定默认dns
- `strategy`解析策略 prefer_ipv4表示优先ipv4，only_ipv4表示仅使用ipv4
### Inbounds (入站)：
针对抗封锁，目前最主流的是 VLESS + XTLS-Reality。
**VLESS + Reality**：VLESS 处理代理连接，Reality 使用配置的握手目标、公钥和 Short ID 建立连接并提供回退行为；具体效果取决于目标站点与网络环境，不涉及窃取目标站点证书。
```json
"inbounds": [
{
	"type": "vless",
	"type": "vless-in",
	"listen": "::",
	"listen_port": 443,
	"users": [
	{
		"name": "Your_name",
		"uuiid": "你的-UUID-字符串",
		"flow": "xtls-rprx-vision"
	}
	],
	"tls": {
		"enabled": true,
		"server_name": "www.nvidia.com",
		"reality": {
			"enabled": true,
			"handshake": {
				"server": "www.nvidia.com",
				"server_port": 443
			},
			"private_key": "你的-Reality-私钥",
			"short_id": ["你的-ShortID"]
		}
		
	}
}
]
```
1. **基础监听配置 (Base Connection Settings)**
- `type` **协议声明**
- `tag` 为该入站模块分配一个**全局唯一的标识符**
- `listen` **绑定地址** ::表示同时监听全部 IPv4 和 IPv6 地址
- `listen_port` **监听端口**：本示例使用 443，这是 HTTPS 的常用端口；协议本身不要求必须使用 443。
2. **身份鉴权配置 (Users Configuration)**
- `users` **用户列表** 
- `name` **用户标识**，主要用于系统日志的生成。
- `uuid` **通用唯一识别码** (Universally Unique Identifier)。这也是 VLESS 握手阶段唯一需要比对的鉴权信息。
- `flow` **流控算法** (Flow Control)，这是 VLESS 应对 TLS 深度包检测的核心技术。`Vision` 流控主要针对 TLS 1.3 协议设计，它的作用是对内层代理流量的 TLS 记录进行长度填充（Padding）与重组，彻底消除“TLS in TLS（套娃）”的包长特征；并在验证安全后，直接在内核层面拼接（Splice）内外层数据流，极大降低 CPU 拷贝开销。
3. **传输层安全配置 (TLS Configuration)**
- `enabled` **启用 TLS 加密**，由于 VLESS 属于“零加密”协议，必须在传输层之上强制建立 TLS 隧道，以确保数据在公网传输过程中的绝对机密性（Confidentiality）。
- `server_name` **服务器名称指示** (SNI, Server Name Indication)，在 TLS 握手阶段（Client Hello），客户端必须在扩展字段中携带该 SNI 字符串。服务端内核会提取该字段，只有当请求的 SNI 与此处配置的值完全一致时，系统才会继续进入鉴权逻辑，这是防范未经授权探测的第一道防线。
1. **REALITY 动态证书与伪装配置 (XTLS-Reality)**
**REALITY** 是彻底解决服务端 TLS 证书指纹特征的革命性技术。它**无需在服务器上存放真实的域名和证书**，而是直接**从白名单目标动态“借用”合法证书**。
- `enabled` **启用 Reality 模块**。接管传统的 TLS 握手流程。
- `handshake` **握手转发目标**：**`"server": "www.nvidia.com"`** 与 **`"server_port": 443`**： 当遭遇主动探测机构（Active Probing）或不合法的网络扫描时，`sing-box` 内核不会断开连接，而是将该 TCP 流以极低的延迟透明代理（Transparent Proxy）至真正的 Nvidia 官网。探测者将与真正的 Nvidia 服务器完成 TLS 握手，并获得具有高度权威的真实证书。这使得该 VPS 在主动探测机制眼中，表现为一个完全合法的海外商业网站。
- `private_key` **服务端私钥**，Reality 基于 X25519 椭圆曲线算法生成的私钥。只有持有对应公钥的合法客户端，才能在 TLS 握手过程中通过 ECDHE（椭圆曲线迪菲-赫尔曼秘钥交换）派生出正确的代理会话密钥，进入 VLESS 的代理逻辑。
- `short_id` **短标识**：Reality 客户端与服务端需要匹配的十六进制字符串，按当前文档为 0–8 位；它不是独立的会话管理或防重放机制。
### Outbounds (出站)与Endpoints（端点）：
```json
"outbounds": [
{
	"type": "direct",
	"tag": "direct"
},
{
	"type": "block",
	"tag": "block"
}
],
// 所有的 Endpoints（端点）标签，自动且隐式地可以作为 Outbounds（出站）标签在路由模块中被直接调用。
"endpoints": [
{
	"type": "wireguard",
	"tag": "warp-ipv6-out",
	"address": [
		"虚拟网卡ipv4地址/32",
		"虚拟网卡ipv6地址/128"
	],
	"private_key": "虚拟网卡本地私钥",
	"peers": [
		"address": "远端对接服务器IP地址",
		"port": 端口号,
		"public_key": "公钥",
		"allow_ips": ["0.0.0.0/0","::/0"]
	]
}
]
```
1. **出站网络配置 (Outbounds)**
- `direct` **直连出站**，当流量被路由到此节点时，`sing-box` 内核会剥离所有的代理逻辑，直接调用宿主机（VPS）的物理网卡向目标发起 TCP/UDP 连接。
- `block` **阻断出站**。一个虚拟的黑洞通道（Blackhole）。任何被路由到此节点的网络数据包，将直接被系统静默丢弃（Drop），不返回任何错误回执。
- `tag` 全局标识符，供 `route` 模块寻址使用。
2. **网络端点配置 (Endpoints)**
- `type` **协议类型**。`wireguard` 声明该端点为一个基于 UDP 数据封装的 WireGuard 隧道协议接口。
- `tag` **接口名称/标签**。虽然定义在 `endpoints` 模块中，但在 `sing-box` 的架构设计里，它**隐式且自动地注册为一个出站目标（Outbound Target）**，可以直接被 `route` 规则调用。
3. **本地接口属性 (Local Interface Properties)**
- `address` **本地接口 IP（虚拟网卡 IP）**。在示例中这是 Cloudflare WARP 分配给这块虚拟网卡的内网 IP 地址。**双栈机制**：将两个协议簇的 IP 组合在一个数组中，赋予了这块虚拟网卡同时承载 IPv4 和 IPv6 流量的能力（Dual-Stack）。这正是解决 `missing IPv4 local address` 报错的根本所在。
- `private_key` **本地私钥**。基于 Curve25519 算法生成的非对称加密私钥，相当于这块网卡的“身份证”。它是后续与远端服务器进行握手和派生对称会话密钥的基础。
4. **对等节点配置 (Peers Configuration)**
在 WireGuard 协议中，没有传统的“客户端”或“服务端”概念，网络两端的参与者互为对等节点（Peer）。这里的 `peers` 数组定义了**这块虚拟网卡可以与谁进行通信**。
- `address` **对等节点寻址信息**。这是远端 Cloudflare 边缘节点的物理公网 IP 地址和监听端口（通常是 `162.159.192.1:2408`）。你的 VPS 将通过宿主机物理网卡，向这个目标发送被封装和加密后的 UDP 数据包。
- `public_key` **对等节点公钥**。这是 Cloudflare 服务器的公开身份信息（通常是固定的特征字符串）。在建立连接时，本地接口会使用自己的私钥与这个公钥进行极其安全的密钥协商（Noise Protocol Framework）。
- `allowed_ips` **路由投递控制（加密域映射）**。用于控制**哪些目标 IP 的流量，可以被打包发送给这个 Peer**
### Route (路由)：
```json
"route": {
	"default_domain_resolver": "google-dns",
	"rules": [
	{
		"network": ["tcp","udp"],
		"ip_cidr": [
			"127.0.0.0/8",
			"192.168.0.0/16",
			"10.0.0.0/8",
			"172.16.0.0/12"
		],
		"outbound": "block"
	},
	{
		"port": [22],
		"outbound": "direct"
	},
	{
		"network": ["udp"],
		"port": [443],
		"outbound": "block"
	},
	{
		"domain_keyword": [
			"google",
			"youtube",
			"gemini"
		],
		"outbound": "warp-ipv6-out"
	}
	],
	"final": "direct"
}
```
   1. **全局路由配置 (Global Routing Settings)**
- `default_domain_resolver` **默认域名解析器 (Default Domain Resolver)** 指定路由模块在需要执行 IP 匹配规则时，所调用的前置 DNS 解析器。
- `rules` **策略路由规则链**。定义流量调度的核心逻辑。引擎严格按照数组中对象排列的先后顺序进行**自上而下的匹配（Top-down Matching）**。采用短路求值（Short-circuit evaluation）机制，一旦数据包命中某条规则，立即执行转发，跳过后续所有规则。
- `final` **默认出站/兜底路由**。定义默认网关策略。如果一个数据包遍历了 `rules` 数组中的所有规则均未命中，则最终被无条件移交至该标签指定的出站通道。
2. 规则匹配参数标签 (Matching Criteria Tags)
这些标签存在于 `rules` 数组的内部对象中，用于提取数据包的特征并进行条件判断。`sing-box` 支持多条件**逻辑与（AND）**，即如果在同一个大括号 `{}` 内写了多个匹配标签，数据包必须同时满足这些条件才算命中。
**L3/L4 网络与传输层匹配 (Network & Transport Layer)**
- `network` **传输层协议匹配**。精确拦截或放行特定的四层协议。如果不写该标签，默认同时匹配 TCP 和 UDP。
- `port` **目标端口匹配**。基于请求的目标端口号进行拦截或分流计算。基于请求的目标端口号进行拦截或分流计算。
- `ip_cidr` **无类别域间路由匹配 (Classless Inter-Domain Routing)**。对数据包的目标 IP 地址进行极其高效的二进制掩码对齐匹配。通常用于划分局域网（LAN）私有地址块或封禁特定的公网 ASN IP 段。
**L7 应用层与逻辑扩展匹配 (Application Layer & Extension)**
- `domain_keyword` **域名子字符串模糊匹配**。要目标域名中**包含**数组中的任意字符串（例如 `"google"` 可以匹配 `www.google.com`、`api.google.co.jp` 等），即判定为命中。在处理拥有大量复杂子域名或区域后缀的大型互联网企业流量时，效率极高。
- `domain_suffix`**域名后缀严格匹配**。基于域名的层级结构进行从右向左的严格比对。例如 `"google.com"` 将匹配 `google.com` 本身及其所有下级子域名（如 `mail.google.com`），但**绝不会匹配** `fakegoogle.com`。这是防止域名仿冒和精准放行 CDN 静态资源域名的标准手段。
- 旧版的 `geoip` 规则字段已移除；本示例将未命中规则的流量交给 `final: direct`，如需按地区分流，应改用当前版本支持的规则集。
**规则执行动作标签 (Action Tags)**
- `outbound`这是条件判断的**结果输出**。当数据包满足了该规则块内的所有匹配标签（如 `network`、`port`、`domain_keyword` 等）后，路由引擎就会向系统内核下达转发指令：将该流量强行移交至该 `outbound` 标签绑定的物理通道或虚拟隧道中去。
# 二、Sing-box 实践示例
## 第一阶段：环境准备与核心安装 (Environment & Installation)
示例的VPS环境是Debian 12 (Bookworm)
### 1. 更新系统并安装依赖
```bash
sudo apt update && sudo apt upgrade -y
sudo apt install -y curl jq openssl nano

# 使用开源社区公认的极简一键脚本安装wgcf，自动识别系统架构并拉取最新版
bash <(curl -fsSL https://raw.githubusercontent.com/P3TERX/script/master/wgcf.sh)

# 赋予执行权限并移动到系统环境变量目录
chmod +x wgcf
sudo mv wgcf /usr/local/bin/
```
`jq`:一个轻量级、极其灵活的命令行 JSON 处理器。
`openssl`:一个极其强大的开源工具包，主要用于实现安全套接字层 (SSL v2/v3) 和传输层安全 (TLS v1) 网络协议，以及相关的密码学算法。
`nano`:运行在 Linux 终端里的文本编辑器。
`wgcf`:用于向 Cloudflare API 申请 WARP 账户的第三方工具。
[wgcf项目开源地址](https://github.com/ViRb3/wgcf/)
### 2. 安装 sing-box 官方最新版本
使用官方提供的一键脚本进行安装。该脚本会自动识别系统架构并配置 `systemd` 守护进程。
```bash
bash <(curl -fsSL https://sing-box.app/deb-install.sh)
```
## 第二阶段：密码学凭证生成 (Cryptographic Credential Generation)
在编写配置文件之前，我们需要在终端生成 VLESS 和 XTLS-Reality 所需的核心密钥对。请依次执行以下命令，并**将屏幕输出的结果保存在你的本地记事本中备用**。
### 1. 生成 VLESS UUID
这是客户端连接节点的唯一身份凭证。
```bash
sing-box generate uuid
# 输出示例: 5b6c3b... (记为 [YOUR_UUID])
```
### 2. 生成 XTLS-Reality 密钥对
这将生成一对非对称密钥。私钥（Private Key）填入服务器配置，公钥（Public Key）留给手机/电脑客户端使用。
```bash
sing-box generate reality-keypair
# 输出示例: 
# PrivateKey: yN1... (记为 [YOUR_REALITY_PRIVATE_KEY]) 
# PublicKey: 8fT... (记作客户端备用)
```
### 3. 生成 Reality Short ID (简短会话 ID)
生成一个 16 位的十六进制字符串，用于会话复用防重放。
```bash
openssl rand -hex 8
# 输出示例: 1a2b3c4d5e6f7a8b (记为 [YOUR_SHORT_ID])
```
### 4. 申请 WARP 账户并生成配置
通过 `wgcf` 向 Cloudflare API 注册一个免费的设备账户，并提取 WireGuard 隧道所需的本地网络接口参数。
```bash
# 创建一个专用目录存放 WARP 配置文件
mkdir -p ~/warp-config && cd ~/warp-config

# 注册 WARP 账户 (自动接受使用条款)
wgcf register --accept-tos

# 根据注册信息生成标准的 WireGuard 配置文件 (wgcf-profile.conf)
wgcf generate

# 查看 wgcf-profile.conf 文件内容
cat wgcf-profile.conf
# 示例输出，仅展示[Interface]部分:
[Interface]
PrivateKey = xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx=
Address = 172.16.0.2/32
Address = 2606:4700:110:xxxx:xxxx:xxxx:xxxx:xxxx/128
DNS = 1.1.1.1
MTU = 1280
```
**参数提取对照表（wgcf-profile.conf）**
请从上述输出中提取对应数值，填入记事本备用。它们将直接用于后续 `sing-box` 配置中的 `endpoints` 模块：
- 提取 `PrivateKey` 的值 -> 记为 **`[YOUR_WARP_PRIVATE_KEY]`**
- 提取第一行 `Address` 的值 -> 记为 **`[YOUR_WARP_IPV4]`** (通常为 `172.16.0.2`)
- 提取第二行 `Address` 的值 -> 记为 **`[YOUR_WARP_IPV6]`** (通常以 `2606:4700:` 开头)
## 第三阶段：核心配置文件组装 (Configuration Assembly)
`sing-box` 的主配置文件位于 `/etc/sing-box/config.json`。清空默认配置，写入自己的相关配置。
```bash
# 编辑配置文件，并在编辑器中替换原有内容
sudo nano /etc/sing-box/config.json
```
**标准配置文件示例：**
```json
{
  "log": {
    "level": "info",
    "timestamp": true
  },
  "dns": {
    "servers": [
      {
        "tag": "cloudflare-dns",
        "type": "udp",
        "server": "1.1.1.1",
        "detour": "direct"
      },
      {
        "tag": "google-dns",
        "type": "udp",
        "server": "8.8.8.8",
        "detour": "direct"
      },
      {
        "tag": "quad9-dns",
        "type": "udp",
        "server": "9.9.9.9",
        "detour": "direct"
      }
    ],
    "rules": [],
    "final": "cloudflare-dns",
    "strategy": "prefer_ipv4"
  },
  "endpoints": [
    {
      "type": "wireguard",
      "tag": "warp-ipv6-out",
      "address": [
        "虚拟网卡ipv4地址/32",
        "虚拟网卡ipv6地址/128"
      ],
      "private_key": "虚拟网卡本地私钥",
      "peers": [
        {
          "address": "远端对接服务器IP地址",
          "port": 2408,
          "public_key": "公钥",
          "allowed_ips": [
            "0.0.0.0/0",
            "::/0"
          ]
        }
      ]
    }
  ],
  "inbounds": [
    {
      "type": "vless",
      "tag": "vless-in",
      "listen": "::",
      "listen_port": 443,
      "users": [
        {
          "name": "Your_name",
          "uuid": "你的-UUID-字符串",
          "flow": "xtls-rprx-vision"
        }
      ],
      "tls": {
        "enabled": true,
        "server_name": "www.nvidia.com",
        "reality": {
          "enabled": true,
          "handshake": {
            "server": "www.nvidia.com",
            "server_port": 443
          },
          "private_key": "你的-Reality-私钥",
          "short_id": [
            "你的-ShortID"
          ]
        }
      }
    },
    {
      "type": "mixed",
      "tag": "local-test-in",
      "listen": "127.0.0.1",
      "listen_port": 10080
    }
  ],
  "outbounds": [
    {
      "type": "direct",
      "tag": "direct"
    },
    {
      "type": "block",
      "tag": "block"
    }
  ],
  "route": {
    "default_domain_resolver": "cloudflare-dns",
    "rules": [
      {
        "network": [
          "tcp",
          "udp"
        ],
        "ip_cidr": [
          "127.0.0.0/8",
          "192.168.0.0/16",
          "10.0.0.0/8",
          "172.16.0.0/12"
        ],
        "outbound": "block"
      },
      {
        "port": [
          22
        ],
        "outbound": "direct"
      },
      {
        "network": [
          "udp"
        ],
        "port": [
          443
        ],
        "outbound": "block"
      },
      {
        "domain_keyword": [
          "google",
          "youtube",
          "gemini",
          "generativeai",
          "openai",
          "anthropic",
          "claude",
          "tiktok",
          "netflix",
          "disney",
          "spotify",
          "googlevideo"
        ],
        "domain_suffix": [
          "google.com",
          "googleapis.com",
          "gstatic.com",
          "ggpht.com",
          "googleusercontent.com",
          "googlevideo.com",
          "youtube.com",
          "youtube-nocookie.com",
          "ytimg.com",
          "gvt2.com",
          "doubleclick.net",
          "googlesyndication.com",
          "googleadservices.com",
          "googletagmanager.com",
          "2mdn.net",
          "g.co",
          "chatgpt.com",
          "oaistatic.com",
          "oaiusercontent.com",
          "claude.ai",
          "anthropic.com",
          "tiktokv.com",
          "byteoversea.com",
          "nflxvideo.net",
          "netflix.com"
        ],
        "outbound": "warp-ipv6-out"
      }
    ],
    "final": "direct"
  }
}
```
## 第四阶段：语法校验与服务接管 (Validation & Service Control)
配置编写完成后，切忌直接重启服务，必须经过内核的严格校验。
### 1.语法正确性检查
利用 `sing-box` 内置的编译器对 JSON 进行结构校验：
```bash
sudo sing-box check -c /etc/sing-box/config.json
```
如果输出为空或无 `FATAL` 报错，证明语法完全合法。
### 2. 启动服务并设置开机自启
```bash
# 启动 sing-box 服务
sudo systemctl start sing-box

# 设置为系统开机自启
sudo systemctl enable sing-box
```
### 3.监控内核日志
必要时刻可以监控系统日志，观察是否有入站连接和路由命中。此时可以把log中的level改为debug级别。
```bash
sudo journalctl -u sing-box -f
```
