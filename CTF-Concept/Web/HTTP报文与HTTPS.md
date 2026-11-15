---
title: "HTTP 报文与 HTTPS"
date: "2026-11-06T00:39:50+08:00"
tags:
  - CTF
  - Web安全
  - HTTP
  - HTTPS
description: "接续 HTTP 协议基础，拆解请求与响应报文、常见首部字段及 HTTPS 的传输保护机制，并从协议特性出发归纳相关攻击面与加固要点。"
---

> 本文是《HTTP 协议基础》的续篇，讲三件事：HTTP 报文长什么样、首部字段怎么读、HTTPS 如何保护传输；最后补上协议特性对应的攻击面与加固清单。
>
> 想先补协议基础，见《HTTP 协议基础》；想动手做靶场，见《HTTP 实战：CTFHub 入门五题》。

# 一、HTTP 报文结构与首部字段

## 1.1 HTTP 报文长什么样

HTTP 报文本身是由多行数据构成的字符串文本，大致分为**报文首部**和**报文主体**两块，两者由空行划分：

1. **报文首部**：服务器端或客户端处理的请求/响应的内容及属性；
2. **空行**：由 CR（回车符，`0x0d`）和 LF（换行符，`0x0a`）组成，即 `\r\n`；
3. **报文主体**：应被发送的数据。

**请求报文**：

```http
GET /index.html HTTP/1.1
Host: www.example.com
User-Agent: Mozilla/5.0
Accept: text/html,*/*
Cookie: session=abc123

（可选的主体）
```

结构是：

```text
请求行（方法 + URI + 版本）
请求首部字段
通用首部字段
实体首部字段
空行（CRLF）
报文主体
```

**响应报文**：

```http
HTTP/1.1 200 OK
Server: nginx/1.24.0
Date: Wed, 05 Nov 2026 15:00:00 GMT
Content-Type: text/html; charset=utf-8
Content-Length: 1024
Set-Cookie: session=abc123; HttpOnly; Secure

<html>...
```

结构是：

```text
状态行（版本 + 状态码 + 原因短语）
响应首部字段
通用首部字段
实体首部字段
空行（CRLF）
报文主体
```

**几个容易忽略的细节**：

- 首部字段名**不区分大小写**（`Content-Type` 和 `content-type` 等价），但值通常区分；
- 首部字段的顺序一般不重要，但 `Host` 必须出现（HTTP/1.1）；
- 空行是"首部结束、主体开始"的分界，**请求走私**就是利用前后端对这个分界的理解差异；
- `Content-Length` 和 `Transfer-Encoding` 同时出现时，优先级和处理方式不同，这也是走私的经典入口。

## 1.2 首部字段分类

课件把首部分为五类：通用首部、请求首部、响应首部、实体首部、Cookie 首部。下面按类整理，并补上安全含义。

**通用首部字段**（请求和响应都会用）

| 字段 | 作用 | 安全含义 |
| --- | --- | --- |
| `Cache-Control` | 控制缓存机制 | 配置不当导致敏感数据被缓存 |
| `Connection` | 控制连接（如 `close`、`keep-alive`） | 影响持久连接与走私 |
| `Date` | 报文创建时间 | 时间戳分析 |

**请求首部字段**（客户端 → 服务器）

| 字段 | 作用 | 安全含义 |
| --- | --- | --- |
| `Host` | 请求资源的主机和端口 | 虚拟主机依据；Host 头注入 |
| `User-Agent` | 客户端操作系统、浏览器信息 | 可伪造；可能被拼进日志/SQL |
| `Referer` | 访问当前页面的上一个页面 | 可能泄露敏感 URL；可伪造/为空 |
| `Cookie` | 请求者的身份凭证 | 篡改、窃取、CSRF |
| `Accept` | 客户端希望接收的 MIME 类型 | 内容协商 |
| `Accept-Charset` | 客户端接收的字符集 | 编码相关攻击 |
| `Authorization` | 认证凭证（如 Basic） | Base64 可逆；必须配 HTTPS |
| `Origin` | 请求来源（CORS） | 跨域校验依据 |
| `X-Forwarded-For` | 代理链上的客户端 IP | 可伪造，影响 IP 白名单/限速 |

**响应首部字段**（服务器 → 客户端）

| 字段 | 作用 | 安全含义 |
| --- | --- | --- |
| `Server` | Web 服务软件及版本 | 版本泄露，辅助选 CVE |
| `Set-Cookie` | 向客户端设置 Cookie | 属性配置决定安全性 |
| `Last-Modified` | 资源最后修改时间 | 缓存协商 |
| `Content-Length` | 正文长度 | 与 TE 冲突导致走私 |
| `Location` | 引导跳转到不同资源 | 开放重定向 |
| `WWW-Authenticate` | 认证质询 | Basic 认证入口 |
| `ETag` | 资源标识 | 缓存、信息泄露 |
| `Access-Control-Allow-Origin` | CORS 允许的来源 | 配置过宽导致跨域读取 |

**实体首部字段**（描述主体内容）

| 字段 | 作用 |
| --- | --- |
| `Allow` | 通知客户端支持的方法（如 `GET, HEAD`） |
| `Content-Encoding` | 主体采用的编码（如 gzip） |
| `Content-Language` | 主体语言 |
| `Content-Length` | 主体大小 |
| `Content-Location` | 主体对应的实际 URI |
| `Content-Type` | 主体的媒体类型 |

**Cookie 首部字段**

| 字段 | 方向 | 作用 |
| --- | --- | --- |
| `Set-Cookie` | 响应 | 服务器设置 Cookie |
| `Cookie` | 请求 | 客户端回传 Cookie |

## 1.3 Cache-Control 指令

`Cache-Control` 是缓存行为的"总开关"，课件分别列了请求指令和响应指令。

**请求指令**

| 指令 | 含义 |
| --- | --- |
| `no-cache` | 强制向源服务器再次验证 |
| `no-store` | 不缓存请求或响应的任何内容 |
| `max-age=[秒]` | 响应的最大 Age 值 |

**响应指令**

| 指令 | 含义 |
| --- | --- |
| `public` | 所有内容都可被缓存（客户端和代理都可） |
| `private` | 只缓存到私有缓存（仅客户端，代理不可） |
| `no-cache` | 必须先与服务器确认响应是否被更改，才能用于后续请求；有合适的验证令牌时会发起往返验证 |
| `no-store` | 所有内容都不缓存到缓存或 Internet 临时文件 |
| `must-revalidate` / `proxy-revalidate` | 缓存失效后必须回源验证 |
| `max-age=xxx` | 内容在 xxx 秒后失效；HTTP/1.1 可用，与 `Last-Modified` 一起使用时优先级较高 |

**安全视角**：包含敏感数据的响应应设置 `Cache-Control: no-store`（或 `no-cache, private`），否则可能被浏览器、代理、CDN 缓存，导致他人通过共享设备或缓存节点读到别人的数据。

## 1.4 首部字段里的攻击面

首部是 HTTP 里"信息密度最高"的部分，也是攻击者最喜欢动手脚的地方：

**（1）Host 头注入**

很多应用用 `Host` 生成链接（找回密码、支付回调、跳转地址）。如果直接信任 `Host`：

```http
GET /reset HTTP/1.1
Host: evil.com
```

服务器可能生成 `https://evil.com/reset?token=...` 的邮件，受害者一点，token 就泄露了。Host 头还可能被用于缓存投毒、SSRF、绕过虚拟主机限制。

**防御**：配置允许的 Host 白名单，不信任客户端传入的 Host。

**（2）X-Forwarded-For 伪造**

应用如果直接用 `X-Forwarded-For` 判断客户端 IP，攻击者可以伪造：

```http
X-Forwarded-For: 127.0.0.1
```

从而绕过 IP 白名单、限速、审计。**防御**：只信任来自可信代理的 XFF，并从右往左取真实 IP。

**（3）Referer 校验绕过**

有些系统用 `Referer` 判断请求是否来自本站。攻击者可以：

- 直接不带 `Referer`（某些浏览器/场景下允许）；
- 用 `https://target.com.evil.com` 这种"前缀包含"的域名；
- 用 `https://evil.com/?target.com` 这种"包含"的 URL。

**防御**：解析 Referer 的 host 做精确匹配，而不是字符串包含；敏感操作加 CSRF Token。

**（4）User-Agent 注入**

UA 被写进日志、数据库、管理后台时，可能造成：

- SQL 注入（`User-Agent` 拼进查询）；
- 存储型 XSS（后台展示 UA 时未转义）；
- 日志注入（换行符伪造日志行）。

**（5）CRLF 注入 / 响应拆分**

如果应用把用户输入直接拼进响应头，攻击者可以注入 `%0d%0a` 来"提前结束"当前首部，伪造新的首部甚至响应体：

```http
Location: https://example.com/%0d%0aSet-Cookie:%20admin=1
```

**防御**：对进入首部的数据做严格过滤，禁止 `\r`、`\n`。

**（6）请求走私（Request Smuggling）**

当前端代理（如 Nginx）和后端服务器（如 Tomcat）对 `Content-Length` 与 `Transfer-Encoding` 的优先级理解不一致时，攻击者可以构造一个"前端认为是一个请求、后端认为是两个请求"的报文，从而绕过访问控制、污染缓存、劫持他人请求。

经典组合：

- **CL.TE**：前端用 `Content-Length`，后端用 `Transfer-Encoding`；
- **TE.CL**：前端用 `Transfer-Encoding`，后端用 `Content-Length`；
- **TE.TE**：两端都支持 TE，但通过混淆（如 `Transfer-Encoding: xchunked`）让其中一方忽略。

**防御**：代理与后端使用一致的解析库；拒绝同时出现 CL 和 TE 的请求；统一规范化。

**（7）Basic 认证**

`Authorization: Basic base64(user:pass)` 的本质是"明文 + 可逆编码"，不是加密。它的问题：

- Base64 可逆，抓包即可还原；
- 没有登出机制（浏览器会一直带着凭证）；
- 通常配合 401 + `WWW-Authenticate` 使用，容易遭到暴力破解；
- 必须配合 HTTPS，否则等于明文传输。

**防御**：Basic 认证只用于内网或临时场景，且必须走 HTTPS；生产环境用更安全的认证方式（OAuth2、OIDC、Session + MFA），并做速率限制与锁定。

---

# 二、HTTPS：传输层安全

## 2.1 HTTP 的三大风险

课件用一句话概括了 HTTP 的问题：**窃听、篡改、冒充**。

| 风险 | 说明 | 后果 |
| --- | --- | --- |
| 窃听 | 明文传输，中间人可抓包 | 密码、Cookie、身份证号泄露 |
| 篡改 | 中间人可修改请求/响应 | 插入广告、恶意脚本、转账改账号 |
| 冒充 | 无法验证对方身份 | 钓鱼站、假冒服务器 |

HTTPS 的公式很简单：

```text
HTTPS = HTTP + SSL/TLS
```

默认端口从 80 变成 443。HTTP 定义应用层语义，TLS 为传输提供机密性、完整性与身份认证。

## 2.2 密码学组件

TLS 不是一种算法，而是一套算法组合。理解它，需要掌握四个组成部分：

| 机制 | 作用 | 在 TLS 中的角色 |
| --- | --- | --- |
| 对称加密（AES、ChaCha20） | 加解密快 | 加密真正的业务数据 |
| 非对称加密（RSA、ECDHE） | 加解密慢，但能安全交换密钥 | 协商出对称密钥 |
| 哈希（SHA-256） | 单向摘要，防篡改 | 完整性校验、证书签名 |
| 证书 + CA | 绑定"公钥 ↔ 身份" | 防冒充 |

核心思路是：**用慢的非对称加密，安全地协商出一个快的对称密钥；之后用对称密钥加密数据。**

## 2.3 TLS 1.2 协商过程

课件把 TLS 1.2 的握手拆成五步，下面用时序图表示：

```text
客户端                                            服务器
  |                                                |
  | 1. ClientHello                                 |
  |    client_random、支持的加密套件、SNI、ALPN     |
  | ---------------------------------------------> |
  |                                                |
  | 2. ServerHello                                 |
  |    server_random、选定的加密套件                |
  | <--------------------------------------------- |
  |    Certificate（服务器证书链）                  |
  | <--------------------------------------------- |
  |    ServerKeyExchange（ECDHE 参数 + 签名）       |
  | <--------------------------------------------- |
  |    ServerHelloDone                             |
  | <--------------------------------------------- |
  |                                                |
  | 3. ClientKeyExchange（客户端 ECDHE 公钥）       |
  | ---------------------------------------------> |
  |    ChangeCipherSpec                            |
  | ---------------------------------------------> |
  |    Finished（加密的握手校验）                   |
  | ---------------------------------------------> |
  |                                                |
  | 4. NewSessionTicket                            |
  | <--------------------------------------------- |
  |    ChangeCipherSpec                            |
  | <--------------------------------------------- |
  |    Finished                                    |
  | <--------------------------------------------- |
  |                                                |
  | 5. Application Data（加密的 HTTP 报文）         |
  | <============================================> |
```

**第 1 步：ClientHello**

ClientHello 报文携带客户端支持的版本、加密套件和扩展，其中三个关键随机要素是：

- `client_random`：32 字节随机数，后续参与密钥推导；
- `cipher_suites`：支持的加密套件列表；
- 扩展：`SNI`（客户端请求的域名）、`ALPN`（客户端希望协商的 HTTP 版本）、`supported_groups`（支持的椭圆曲线）。

**第 2 步：ServerHello + Certificate + ServerKeyExchange + ServerHelloDone**

服务器通过 ServerHello、Certificate、ServerKeyExchange、ServerHelloDone 返回选定的版本、加密套件、证书和临时公钥：

- `server_random`：另一个 32 字节随机数；
- `Certificate`：服务器证书链，客户端用它验证"这个公钥确实属于这个域名"；
- `ServerKeyExchange`：使用 ECDHE 时，服务器在这里给出曲线参数、临时公钥，并用证书私钥签名，防止被篡改；
- `ServerHelloDone`：表示服务器端握手消息发送完毕。

**第 3 步：ClientKeyExchange + CCS + Finished**

客户端：

- 生成自己的 ECDHE 临时密钥对，将公钥放入 `ClientKeyExchange`；
- 双方通过 ECDHE 协商得到同一个**预主密钥**（pre-master secret）；
- 发送 `ChangeCipherSpec`，通知对端后续报文将使用协商出的密钥加密；
- 发送 `Finished`，其中包含对整个握手过程的校验值，并以协商密钥加密传输。

**第 4 步：服务器切换加密并确认**

服务器也发送 `ChangeCipherSpec` 和 `Finished`，握手完成。

**密钥推导**

```text
pre-master secret（ECDHE 协商出的共享密钥）
        ↓ PRF
master secret = PRF(pre-master, "master secret", client_random + server_random)
        ↓ PRF
key block = PRF(master_secret, "key expansion", server_random + client_random)
        ↓ 切分
client_write_key / server_write_key / client_write_IV / server_write_IV / ...
```

这里有三个关键点：

1. **临时密钥**：ECDHE 每次握手都生成新的密钥对，即使服务器的长期私钥泄露，攻击者也无法解密历史流量（前向保密）；
2. **随机数**：`client_random` 和 `server_random` 保证每次会话的密钥不同；
3. **证书签名**：`ServerKeyExchange` 被证书私钥签名，中间人改不了。

**第 5 步：Application Data**

握手完成后，双方用协商出的对称密钥加密 HTTP 报文。从抓包看，后面的内容全是"Application Data"，无法直接读出明文。

## 2.4 TLS 1.3 的改进

TLS 1.3（2018 年，RFC 8446）做了大幅简化：

- **1-RTT 握手**：客户端第一次就能猜出密钥协商参数，减少一个来回；
- **只保留 (EC)DHE**：删掉了 RSA 密钥交换、静态 DH 等不安全/不提供前向保密的方案；
- **只保留 AEAD**：删掉了 CBC 模式、RC4、SHA-1 等弱算法；
- **加密证书**：服务器证书在 TLS 1.3 里也是加密传输的，减少信息泄露；
- **0-RTT 恢复**：会话恢复时可以"零往返"发数据，但存在重放风险，需要应用层配合。

## 2.5 搭建 TLS 明文分析环境

课件给了一个很实用的实验：用 `SSLKEYLOGFILE` 让浏览器把 TLS 会话密钥写进文件，再让 Wireshark 用这个文件解密流量。

**步骤一：设置环境变量**

Windows 下新建系统变量：

```text
变量名：SSLKEYLOGFILE
变量值：C:\tls\sslkey.log
```

**步骤二：让 Wireshark 读取密钥文件**

打开 Wireshark → 编辑 → 首选项 → 协议 → TLS → 在 `(Pre)-Master-Secret log filename` 中填入上面的文件路径。

**步骤三：抓包分析**

用 Chrome 访问 HTTPS 网站并抓包。此时 Wireshark 能把 TLS 的 Application Data 解密为 HTTP 明文，可直接查看请求行、首部和响应体。

**注意**：

- 环境变量要在启动浏览器**之前**设置；
- 该方法只对"你自己机器上的浏览器会话"有效，不能解密别人的流量；
- 生产环境不要长期开启，密钥文件本身就是敏感信息。

## 2.6 HTTPS 的安全视角

**HTTPS 解决了什么**

- 机密性：中间人看不到明文；
- 完整性：中间人改不了内容；
- 身份认证：证书链验证服务器身份。

**HTTPS 没解决什么**

- 钓鱼网站也可以用 HTTPS，锁图标不代表"网站可信"；
- 服务器自身有漏洞，HTTPS 救不了；
- 证书配置错误（过期、域名不匹配、自签名）会让用户习惯性点击"继续访问"；
- 中间人代理（Burp、企业 SSL 解密设备）可以通过安装根证书来解密流量。

**常见攻击与防御**

| 攻击 | 说明 | 防御 |
| --- | --- | --- |
| 中间人 | 伪造证书劫持流量 | 证书校验、HSTS、证书锁定 |
| 降级 | 把 HTTPS 降成 HTTP | HSTS、301 跳转、预加载列表 |
| 证书伪造 | 攻击者控制 CA 或 CA 被攻破 | 证书透明（CT）、CAA 记录、监控 |
| 剥离 | 把页面里的 HTTPS 链接改成 HTTP | 混合内容拦截、CSP `upgrade-insecure-requests` |
| 会话劫持 | 窃取 Cookie 后重放 | `Secure`、`HttpOnly`、短期会话、绑定设备 |

**给开发者的检查清单**：

- 全站 HTTPS，HTTP 301 跳转；
- 启用 HSTS，`max-age` 足够长；
- 使用现代 TLS 配置（TLS 1.2+，禁用弱套件）；
- 证书自动续期与到期监控；
- Cookie 加 `Secure`、`HttpOnly`、`SameSite`；
- 不在 URL 里传敏感参数；
- 对混合内容做拦截。

---

# 三、协议特性的攻击面：八类滥用方式

前面几节从安全角度分析了 HTTP 的各个组成部分。这一节按协议元素逐项分析其攻击面：**HTTP 的每个组成部分都可能成为攻击输入。**

## 3.1 方法：绕过仅校验 GET/POST 的防护

很多 WAF、路由、权限中间件只对 `GET`、`POST` 做检查。攻击者换一个方法，可能就绕过去了：

```http
CTFHUB /index.php HTTP/1.1
Host: target
```

只要后端框架接受任意方法 token，请求就会被正常处理；而 WAF 规则通常未覆盖 `CTFHUB` 这一方法名。

类似的还有：

- `PUT` 写文件（WebDAV、Tomcat CVE-2017-12615）；
- `DELETE` 删资源；
- `TRACE` 回显 Cookie；
- `OPTIONS` 探测 `Allow`；
- `PATCH` 批量赋值；
- `X-HTTP-Method-Override: PUT` 用 POST 冒充 PUT；
- 大小写变形（`get`、`Get`）、加空格、加 `%20`，观察哪一层执行了规范化。

**防御**：在反向代理层做**方法白名单**，只放行业务真正需要的方法；后端框架也做一次校验，不要依赖单层。

## 3.2 URL 与参数：注入类漏洞的主要入口

- **SQL 注入**：`?id=1'`；
- **XSS**：`?q=<script>...</script>`；
- **命令注入**：`?ip=127.0.0.1;id`；
- **路径穿越**：`?file=../../etc/passwd`；
- **SSRF**：`?url=http://127.0.0.1:6379/`；
- **开放重定向**：`?next=https://evil.com`；
- **参数污染（HPP）**：`?id=1&id=2`，WAF 取第一个、后端取最后一个，就能绕过。

**防御**：参数化查询、白名单校验、URL 解析后再校验、对重定向目标做域名白名单。

## 3.3 状态码与跳转：对响应的错误信任

- **302 开放重定向**：登录成功后跳回 `next` 参数，攻击者把 `next` 改成钓鱼站；
- **302 认证绕过**：有些系统仅根据状态码判断，非 401 即视为认证通过，攻击者可构造返回 302 的路径；
- **304 缓存欺骗**：让用户缓存一个需要权限的页面，再诱导他人访问缓存；
- **403/404 枚举**：根据状态码差异画出目录结构；
- **500 报错注入**：让 SQL 报错出现在错误页里。

**防御**：跳转目标白名单；错误页统一；不要在状态码上做"隐式信任"。

## 3.4 首部：来自客户端的不可信输入

Host、X-Forwarded-For、Referer、User-Agent、Origin、Authorization……每一个都来自客户端，**默认都不可信**。

- **Host 头注入**：密码重置链接被投毒、缓存投毒、SSRF；
- **XFF 伪造**：绕过 IP 白名单、限速、审计；
- **Referer 绕过**：用"包含"关系伪造来源；
- **UA 注入**：日志注入、SQL 注入、存储型 XSS；
- **CRLF 注入**：`%0d%0a` 拆分响应；
- **请求走私**：CL/TE 解析差异；
- **缓存投毒**：利用未被纳入缓存键的首部（如 `X-Forwarded-Host`）。

**防御**：首部白名单、严格校验、代理层规范化、WAF + 后端双重校验。

## 3.5 Cookie 与认证：客户端数据的不可信性

- **Cookie 篡改**：`admin=0` 改成 `admin=1`；
- **Cookie 窃取**：XSS 读 `document.cookie`；
- **会话固定**：攻击者预设 Session ID；
- **CSRF**：浏览器自动带 Cookie；
- **Basic 认证爆破**：Base64 可逆，密码可枚举；
- **JWT 攻击**：`alg=none`、弱密钥、未校验签名、`kid` 注入。

**防御**：服务端会话、Cookie 签名、`HttpOnly`/`Secure`/`SameSite`、CSRF Token、JWT 强校验、登录限速与 MFA。

## 3.6 Body 与编码：解析器差异

- **Content-Type 混淆**：WAF 按表单解析，后端按 JSON 解析；
- **分块传输绕过**：WAF 不解码 chunked，后端解码后执行；
- **参数污染**：同名参数在不同层取不同的值；
- **multipart 边界混淆**：伪造 boundary 让解析器"看错"；
- **字符集混淆**：UTF-7、UTF-16、全角字符绕过关键词匹配。

**防御**：统一解析库、严格 Content-Type 校验、拒绝异常编码、WAF 与后端行为对齐。

## 3.7 连接：持久连接的风险

- **请求走私**：利用 CL/TE 不一致，将第二个请求嵌入第一个请求的报文中；
- **Slowloris**：极慢地发送首部，占满连接；
- **慢速 POST**：极慢地发送 body；
- **连接池污染**：把恶意请求留在连接池里，等下一个用户复用。

**防御**：连接超时、首部/body 大小限制、CL/TE 冲突拒绝、代理与后端一致性。

## 3.8 响应：信息输出过多

- **源码泄露**：HTML 注释、JS 里的接口和密钥、Source Map；
- **报错信息**：堆栈、SQL、路径、版本；
- **目录列表**：服务器把文件列出来；
- **备份文件**：`.git`、`.svn`、`.bak`；
- **Server 首部**：中间件版本；
- **调试接口**：`/actuator`、`/debug`、`/phpinfo.php`。

**防御**：统一错误页、关闭目录列表、移除调试接口、清理源码注释、隐藏版本号。

## 3.9 攻击面汇总

| 协议元素 | 典型攻击 | 关键防御 |
| --- | --- | --- |
| 方法 | 任意方法绕过、PUT 上传、TRACE 回显 | 方法白名单 |
| URL/参数 | 注入、遍历、SSRF、开放重定向、HPP | 参数化、白名单、解析后校验 |
| 状态码 | 302 绕过、枚举、报错注入 | 统一错误处理、跳转白名单 |
| 首部 | Host 注入、XFF 伪造、CRLF、走私 | 首部校验、代理规范化 |
| Cookie | 篡改、窃取、固定、CSRF | 签名、HttpOnly、SameSite、Token |
| Body | 编码混淆、分块绕过、参数污染 | 统一解析、严格类型 |
| 连接 | 走私、慢速攻击、连接池污染 | 超时、大小限制、一致性 |
| 响应 | 源码/报错/版本/目录泄露 | 最小化输出 |

---

# 四、HTTP 层加固清单

如果你是开发、运维或安全工程师，下面这份清单可以直接拿去对照。

## 4.1 方法

- 只放行业务需要的方法（通常 GET/POST/HEAD）；
- 在反向代理层禁用 PUT、DELETE、TRACE、CONNECT；
- 对不支持的方法返回 405，而不是 200 或 500；
- 不要依赖 `X-HTTP-Method-Override` 这类"方法覆盖"机制，除非明确需要。

## 4.2 首部

- 强制 HTTPS，启用 HSTS；
- 配置安全响应头：
  - `Content-Security-Policy`：限制脚本、样式、frame 来源；
  - `X-Content-Type-Options: nosniff`：禁止 MIME 嗅探；
  - `X-Frame-Options: DENY` 或 CSP `frame-ancestors`：防点击劫持；
  - `Referrer-Policy: strict-origin-when-cross-origin`：控制 Referer 泄露；
  - `Permissions-Policy`：限制摄像头、麦克风、地理位置等；
- 隐藏或最小化 `Server`、`X-Powered-By` 等版本信息；
- 校验 `Host`，配置允许的域名白名单；
- 只信任来自可信代理的 `X-Forwarded-For`；
- 对进入响应首部的数据过滤 `\r`、`\n`，防 CRLF 注入。

## 4.3 Cookie

- 会话 Cookie 加 `HttpOnly`、`Secure`、`SameSite=Lax/Strict`；
- 使用 `__Host-` 前缀，限制 Domain/Path；
- 不在 Cookie 里存权限字段、明文密码、敏感信息；
- 登录后重新生成 Session ID，登出后服务端失效；
- 设置合理的过期时间，支持"记住我"时使用独立令牌。

## 4.4 认证

- 全站 HTTPS，禁止 Basic 认证走明文；
- 登录失败限速、锁定、验证码；
- 强制 MFA（管理员、敏感操作）；
- 密码存储用 bcrypt/scrypt/Argon2，加盐；
- 检测弱密码、已泄露密码；
- 统一错误提示，防止账户枚举；
- 记录登录日志，异常登录告警。

## 4.5 输入与输出

- 所有参数做类型、长度、格式校验；
- 数据库访问使用参数化查询/ORM；
- 输出按上下文编码（HTML/JS/URL/CSS）；
- 文件上传白名单 + 重命名 + 禁止执行；
- 文件包含用白名单映射；
- 命令执行避免拼接，使用参数数组或转义。

## 4.6 代理与网关

- 拒绝同时带 `Content-Length` 和 `Transfer-Encoding` 的请求；
- 统一前后端对报文边界的解析；
- 限制首部大小、body 大小、请求行长度；
- 设置连接超时，防 Slowloris；
- 缓存键要包含所有影响响应的首部，防缓存投毒；
- WAF 规则要覆盖非 GET/POST 方法、分块编码、编码变形。

## 4.7 错误与日志

- 统一错误页，不返回堆栈、SQL、路径；
- 关闭目录列表、调试接口、`phpinfo`；
- 记录方法、路径、状态码、UA、IP、耗时；
- 对 401/403/405/500 的异常增长告警；
- 日志集中存储、防篡改、保留足够时间。

## 4.8 一张检查表

| 检查项 | 是/否 |
| --- | --- |
| 全站 HTTPS + HSTS | ☐ |
| 只开放必要的方法 | ☐ |
| Host 白名单 | ☐ |
| Cookie 三件套（HttpOnly/Secure/SameSite） | ☐ |
| 登录限速 + MFA | ☐ |
| 参数化查询 | ☐ |
| 输出编码 + CSP | ☐ |
| 上传白名单 + 禁止执行 | ☐ |
| 统一错误页 | ☐ |
| 日志与告警 | ☐ |
| 依赖定期更新 | ☐ |

---

# 五、总结

- **报文** = 起始行 + 首部 + 空行 + 主体；CRLF 是首部与主体的边界，也是请求走私的入口。
- **首部** 是输入面：Host、X-Forwarded-For、Referer、User-Agent、Cookie、Authorization 默认都不可信。
- **HTTPS** 用"非对称协商 + 对称加密 + 证书"解决窃听、篡改、冒充，但不解决应用漏洞。
- **防守** 要在方法、首部、Cookie、认证、代理、日志上同时设卡，单点防护没有意义。
