---
title: "HTTP 实战：CTFHub 入门五题"
date: "2026-11-14T13:57:56+08:00"
tags:
  - CTF
  - CTFHub
  - HTTP
description: "通过 CTFHub 五道 HTTP 入门题练习请求方法、重定向、Cookie、基础认证和响应源码查看，记录请求构造、现象观察与解题思路。"
---

> 本文是《HTTP 协议基础》和《HTTP 报文与 HTTPS》的配套实战篇，用 CTFHub 的五道 HTTP 入门题，把请求行、状态码、Cookie、认证、响应体这些知识点落到手上。
>
> 靶场地址和 flag 是动态生成的，文中出现的域名与 flag 仅作示例；所有操作仅针对 CTFHub 提供的合法靶场环境，请勿对未授权的目标进行测试。

![HTTP 实战：CTFHub 入门五题：总览](https://raw.githubusercontent.com/Lehuang0212/Myblog-images/main/images/http/00-overview.png)

# 一、准备工作：为什么自己写请求脚本

浏览器的默认行为会干扰协议测试：

- 会自动跟随 302 跳转，原始响应不会显示；
- 会自动带上 Cookie，不方便观察 `Set-Cookie`；
- 会规范化请求方法，不好发自定义方法；
- 会隐藏原始请求头，调试不方便。

Burp Suite 当然可以，但用 Python 写一个几十行的小脚本，更能看清"HTTP 就是一段文本"这件事。下面是我用的 `http_req.py`，代码按原报告保留；其中 `base64_encode` 来自原报告引用的本地 `codec.py`：

```python
import sys,http.client

from codec import base64_encode

def send_once(host,port,method,path,headers):
    connect = http.client.HTTPConnection(host,port,timeout=5)
    try:
        connect.request(method,path,body="",headers=headers)
        response = connect.getresponse()
        status = response.status
        text = response.read().decode("utf-8","ignore")
        return status, response.headers.items(), text
    except Exception as e:
        print("Error:",e)
        return 0, [], ""
    finally:
        connect.close()

def list_auth(host,port,method,path,user,wordlist):
    with open(wordlist,encoding="utf-8") as f:
        pwds = f.read().splitlines()

    for i,pwd in enumerate(pwds,1):
        token = base64_encode(f"{user}:{pwd}")
        headers = {"Authorization":"Basic " + token}
        status, _, text = send_once(host, port, method, path, headers)
        if  status == 200:
            print(f"[{i}/{len(pwds)}] 命中: {pwd}")
            print(text)
            return pwd,headers
        else:
            print(f"[{i}/{len(pwds)}] {pwd} 失败")

    return "",{}

if __name__ == "__main__":
    if __name__ == "__main__":
        # ---------- 解析命令行 ----------
        pos = []  # 位置参数：域名 端口 方法 路径，按顺序收
        headers = {}  # -H 收集到的请求头
        wordlist = None  # -w 指定的密码文件

        i = 1  # 从 1 开始，跳过 argv[0]（脚本名本身）
        while i < len(sys.argv):  # 遍历所有参数
            arg = sys.argv[i]  # 当前这个参数
            if arg in ("-H", "--header"):  # 遇到 -H → 下一个就是请求头
                name, value = sys.argv[i + 1].split(":", 1)  # 只按第一个冒号切
                headers[name.strip()] = value.strip()  # 收进字典
                i += 2  # 一次吃掉两个（-H 和它的值）
            elif arg in ("-w", "--wordlist"):  # 遇到 -w → 下一个就是字典路径
                wordlist = sys.argv[i + 1]
                i += 2
            else:  # 其余都当位置参数
                pos.append(arg)
                i += 1

        if len(pos) < 4:  # 位置参数不够 4 个就是用法错
            print('用法: python http_req.py <域名> <端口> <方法> <路径> [-H "名: 值"]... [-w 密码文件]')
            sys.exit(0)

        HOST = pos[0]  # 域名
        PORT = int(pos[1])  # 端口
        METHOD = pos[2].upper()  # 方法
        PATH = pos[3]  # 路径

        # ---------- 分流 ----------
        if wordlist:  # 给了 -w → 爆破模式
            pwd, auth_headers = list_auth(HOST, PORT, METHOD, PATH, "admin", wordlist)
            print("正确密码:", pwd)
        else:  # 没给 -w → 单次请求
            status, resp_headers, text = send_once(HOST, PORT, METHOD, PATH, headers)
            print(status)
            for k, v in resp_headers:
                print(f"{k}: {v}")
            print(text)
```

用法：

```bash
python http_req.py <域名> <端口> <方法> <路径> [-H "名: 值"]... [-w 密码文件]
```

这个脚本有三个关键设计：

1. **方法可任意指定**：`http.client` 不会限制方法名，传 `CTFHUB` 就发 `CTFHUB`；
2. **不自动跟随跳转**：`http.client` 收到 302 后不会访问 `Location`；
3. **首部完全可控**：`-H` 可以加任意首部，方便测 Cookie、Authorization、Host。

# 二、第一题：请求方式

![二、第一题：请求方式：请求方式、题目](https://raw.githubusercontent.com/Lehuang0212/Myblog-images/main/images/http/01-method-task.png)

**题目提示**

![二、第一题：请求方式：请求方式、hint](https://raw.githubusercontent.com/Lehuang0212/Myblog-images/main/images/http/02-method-hint.png)

`CTF**B` 是将 `CTFHUB` 中间的 `HU` 替换为 `**` 后的写法。题目考查的是：**HTTP 方法不只有 GET/POST，服务器可能接受任意方法 token，而防护通常只覆盖常见方法。**

**操作**

```bash
python.exe .\http_req.py challenge-b59bb5cb4db5ac2e.sandbox.ctfhub.com 10800 CTFHUB /index.php
```

![二、第一题：请求方式：请求方式、结果](https://raw.githubusercontent.com/Lehuang0212/Myblog-images/main/images/http/03-method-result.png)

**知识点复盘**

- 请求行是 `方法 SP 路径 SP 版本 CRLF`，方法是一个 token，服务端框架通常按字符串匹配；
- 如果服务器只对 GET/POST 做业务处理，其他方法可能落到默认分支或直接放行；
- 这题在真实场景里的对应物是：**WAF 只拦 GET/POST，攻击者换方法绕过**；
- 提示里说"405 就请求 index.php"，是因为有些环境的默认首页不是 `index.php`，请求错路径会得到 `405 Method Not Allowed`，容易误判。

# 三、第二题：302 跳转

![三、第二题：302 跳转：302 跳转、题目](https://raw.githubusercontent.com/Lehuang0212/Myblog-images/main/images/http/04-302-task.png)

**题目提示**

![三、第二题：302 跳转：302 跳转、页面](https://raw.githubusercontent.com/Lehuang0212/Myblog-images/main/images/http/05-302-page.png)

**分析**

用脚本直接请求 `index.php`，**不要跟随跳转**：

```bash
python.exe .\http_req.py challenge-baa98fd2065fdbcd.sandbox.ctfhub.com 10800 GET /index.php
```

**结果**

```text
302
ctfhub{4d6670af664f688e5148237d}
```

状态码是 302，响应体里直接带着 flag。

**为什么浏览器看不到**

浏览器收到 302 后，会自动读取 `Location` 首部，再发一次请求去新地址。用户看到的是"跳转后的页面"，302 这一次的响应体被丢掉了。所以：

- 用浏览器访问，看不到 flag；
- 用 `curl` 时如果加了 `-L`，也会跟随跳转；不加 `-L` 才能看到 302 本身；
- 用 `http.client` 或 Burp 看原始响应，才能看到 302 的 body。

对比一下：

```bash
curl -i  http://target/index.php    # 只看第一次响应，能看到 302 和 body
curl -iL http://target/index.php    # 跟随跳转，看到最终页面
```

**知识点复盘**

- 302 是临时重定向，`Location` 指向新地址；
- 重定向响应也可以带 body，浏览器通常会忽略它；
- 安全测试中，**关闭自动跳转**是基本要求，否则会漏掉大量信息；
- 真实场景里，302 常被用于开放重定向、OAuth 授权码窃取、登录后跳转绕过。

# 四、第三题：Cookie

![四、第三题：Cookie：Cookie、题目](https://raw.githubusercontent.com/Lehuang0212/Myblog-images/main/images/http/06-cookie-task.png)

**题目提示**

![四、第三题：Cookie：Cookie、guest](https://raw.githubusercontent.com/Lehuang0212/Myblog-images/main/images/http/07-cookie-guest.png)

服务端根据请求携带的身份标识判定当前用户为 guest——先看响应头。

**分析**

请求 `index.php`：

```bash
python.exe .\http_req.py challenge-177ac990109d7be0.sandbox.ctfhub.com 10800 GET /index.php
```

![四、第三题：Cookie：Cookie、setcookie](https://raw.githubusercontent.com/Lehuang0212/Myblog-images/main/images/http/08-cookie-setcookie.png)

服务端用 Cookie 里的 `admin` 字段表示"是不是管理员"，当前值是 0。既然是客户端保存的，那就可以改。

**操作**

带上伪造的 Cookie 再请求一次：

```bash
python.exe .\http_req.py challenge-177ac990109d7be0.sandbox.ctfhub.com 10800 GET /index.php -H "Cookie: admin=1"
```

![四、第三题：Cookie：Cookie、结果](https://raw.githubusercontent.com/Lehuang0212/Myblog-images/main/images/http/09-cookie-result.png)

**知识点复盘**

- Cookie 存在客户端，用户拥有完全控制权，**任何"把权限写在 Cookie 里"的设计都是不安全的**；
- 服务端不能信任 Cookie 的内容，除非它经过签名/加密，且服务端能验证；
- 正确做法：Cookie 只存 Session ID，权限判断在服务端查；
- 即使是 Session，也要防固定、防窃取，并设置 `HttpOnly`、`Secure`、`SameSite`。

# 五、第四题：基础认证

![五、第四题：基础认证：基础认证、题目](https://raw.githubusercontent.com/Lehuang0212/Myblog-images/main/images/http/10-basic-task.png)

**题目提示**

访问页面，浏览器弹出登录框——这是 HTTP Basic 认证的典型表现。响应头是：

```http
HTTP/1.1 401 Unauthorized
WWW-Authenticate: Basic realm="..."
```

题目给了一个密码字典附件，一共 100 条常见弱口令，用户名是 `admin`。

![五、第四题：基础认证：基础认证、页面](https://raw.githubusercontent.com/Lehuang0212/Myblog-images/main/images/http/11-basic-page.png)

**分析**

Basic 认证的流程：

1. 客户端请求受保护资源；
2. 服务器返回 401 + `WWW-Authenticate: Basic`；
3. 浏览器弹框，用户输入用户名密码；
4. 浏览器把 `用户名:密码` 用 Base64 编码，放进 `Authorization: Basic <base64>` 重新请求；
5. 服务器解码、比对，成功返回 200，失败再返回 401。

因为 Base64 只是编码不是加密，所以可以写脚本批量尝试。

**操作**

```bash
python.exe .\http_req.py challenge-8de78f5f729ea145.sandbox.ctfhub.com 10800 GET /flag.html -w .\password_list.txt
```

![五、第四题：基础认证：基础认证、结果](https://raw.githubusercontent.com/Lehuang0212/Myblog-images/main/images/http/12-basic-result.png)

**知识点复盘**

- `Authorization: Basic YWRtaW46Y2hlbHNlYQ==` 解码后就是 `admin:chelsea`；
- 第 91 条才命中，说明字典顺序和真实密码分布很重要，实战中常用 rockyou、Top 1000 等字典；
- Basic 认证没有登录失败锁定、没有验证码、没有 MFA，天然适合爆破；
- 传输层如果没有 HTTPS，Base64 等于明文；
- **防御**：强制 HTTPS；限制失败次数并锁定；加验证码/MFA；监控大量 401；不要用 Basic 认证保护敏感系统。

# 六、第五题：响应包源代码

![六、第五题：响应包源代码：源码、题目](https://raw.githubusercontent.com/Lehuang0212/Myblog-images/main/images/http/13-source-task.png)

**题目提示**

flag 位于响应包源代码中——即服务器返回的 HTML 源码。

![六、第五题：响应包源代码：源码、页面](https://raw.githubusercontent.com/Lehuang0212/Myblog-images/main/images/http/14-source-page.png)

**分析**

浏览器只负责"渲染"，不负责"展示源码"。HTML 注释、被 CSS 隐藏的元素、JS 里的常量，用户看不见，但都在响应体里。

**操作**

直接 GET 页面：

```bash
python.exe .\http_req.py challenge-d61fc38d1ca40fbd.sandbox.ctfhub.com 10800 GET /index.html
```

![六、第五题：响应包源代码：源码、结果](https://raw.githubusercontent.com/Lehuang0212/Myblog-images/main/images/http/15-source-result.png)

**知识点复盘**

- 查看源码的方式：浏览器 `view-source:`、F12 → Elements/Sources、`curl`、Burp；
- 真实场景里，源码可能泄露：注释里的测试账号、内网地址、接口路径、API Key、Source Map；
- **防御**：构建时去除注释和 Source Map；不要把敏感信息写进前端；接口鉴权在服务端做，不要靠"前端看不见"。

# 七、五题知识点对照

| 题目 | 对应协议知识点 | 攻击本质 | 关键防御 |
| --- | --- | --- | --- |
| 请求方式 | 请求行、方法 token | 方法白名单缺失，任意方法绕过 | 代理层方法白名单 |
| 302 跳转 | 状态码、`Location` | 只关注跳转结果，忽略原始响应 | 跳转白名单、日志留存 |
| Cookie | `Set-Cookie` / `Cookie` | 客户端可篡改权限字段 | 服务端会话、Cookie 签名 |
| 基础认证 | 401、`WWW-Authenticate`、Base64 | 弱口令 + 可逆编码 | HTTPS、限速、MFA |
| 响应包源代码 | 响应体、HTML 注释 | 信息泄露 | 清理注释、最小化输出 |

做完这五题，可以将它们与协议字段一一对应：

- 方法 → 请求行；
- 302 → 状态行 + `Location`；
- Cookie → 请求/响应首部；
- 基础认证 → 401 + `Authorization`；
- 源代码 → 报文主体。

**HTTP 的每个部分都可以作为测试输入。**

---

# 八、总结

五道题对应五类协议字段，可以总结为：

- **请求方式**：方法不是只有 GET/POST，服务端的"方法白名单"比"方法黑名单"可靠；
- **302 跳转**：状态码和 `Location` 是协议的正常功能，但自动跟随跳转会使测试者看不到原始响应；
- **Cookie**：客户端存储可被任意修改，权限判断必须在服务端完成；
- **基础认证**：Base64 不是加密，弱口令配合无限重试可被暴力破解；
- **响应包源代码**：浏览器展示的是渲染结果，不是服务器返回的全部内容。

这五道题的共同思维是：**观察协议、质疑默认、验证假设**。它们对应的是 HTTP 的请求行、状态行、Cookie 首部、认证首部、响应体——也就是协议最基本的五个组成部分。

协议细节见《HTTP 协议基础》与《HTTP 报文与 HTTPS》。

> 再次提醒：本文所有技术内容仅用于授权测试与防御学习。未经授权对真实系统进行测试，可能触犯法律。
