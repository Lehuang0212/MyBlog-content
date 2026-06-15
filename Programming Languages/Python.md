# Python 基础到项目实战学习笔记

> 本文基于原《Python基础学习记录.md》重新整理，并结合 Python 官方文档、Python Packaging User Guide 以及常见项目实践补充完善。  
> 写作时间：2026-06-15。Python 官网当前最新稳定版本为 Python 3.14.6。日常学习建议使用 Python 3.12+ 或 3.14+；如果学校/课程环境固定为 3.10 或 3.11，本文大多数内容仍然适用。

## 目录

- [1. Python 是什么：先建立整体认识](#1-python-是什么先建立整体认识)
- [2. 环境准备：安装、运行、虚拟环境](#2-环境准备安装运行虚拟环境)
- [3. 基础语法：缩进、注释、标识符与变量](#3-基础语法缩进注释标识符与变量)
- [4. 输入输出：print、input 与字符串格式化](#4-输入输出printinput-与字符串格式化)
- [5. 数据类型：数字、字符串、布尔值与空值](#5-数据类型数字字符串布尔值与空值)
- [6. 运算符与表达式](#6-运算符与表达式)
- [7. 程序结构：分支、循环与模式匹配](#7-程序结构分支循环与模式匹配)
- [8. 容器类型：list、tuple、dict、set](#8-容器类型listtupledictset)
- [9. 函数：把重复逻辑封装起来](#9-函数把重复逻辑封装起来)
- [10. 异常处理：让程序面对错误仍然可控](#10-异常处理让程序面对错误仍然可控)
- [11. 文件读写与上下文管理器](#11-文件读写与上下文管理器)
- [12. 模块、包与导入机制](#12-模块包与导入机制)
- [13. 面向对象编程：类、对象、继承与组合](#13-面向对象编程类对象继承与组合)
- [14. 进阶语法：推导式、迭代器、生成器、装饰器](#14-进阶语法推导式迭代器生成器装饰器)
- [15. 常用标准库路线图](#15-常用标准库路线图)
- [16. 第三方库与生态：学会站在工具之上](#16-第三方库与生态学会站在工具之上)
- [17. 项目实战管理：从脚本到可维护项目](#17-项目实战管理从脚本到可维护项目)
- [18. 小项目示例：命令行待办清单](#18-小项目示例命令行待办清单)
- [19. 学习建议与常见误区](#19-学习建议与常见误区)
- [参考资料](#参考资料)

## 1. Python 是什么：先建立整体认识

Python 是一种解释型、高级、通用编程语言，语法清晰，适合用来学习编程基础，也广泛用于后端开发、自动化脚本、数据分析、人工智能、网络爬虫、测试、运维和桌面工具开发。

Python 的几个重要特点：

1. **代码可读性强**：用缩进表示代码块，不依赖 `{}`。
2. **动态类型**：变量不需要提前声明类型，运行时根据对象决定类型。
3. **强类型**：不同类型不会随便隐式混用，例如字符串 `"1"` 不能直接和整数 `1` 相加。
4. **标准库丰富**：文件、时间、JSON、正则、日志、命令行参数、网络、数据库等常用能力都内置。
5. **第三方生态庞大**：通过 `pip` 可以安装 `requests`、`pandas`、`flask`、`fastapi`、`pytest` 等大量库。

学习 Python 可以按下面这条路线走：

```text
环境安装
  -> 基础语法
  -> 数据类型和容器
  -> 流程控制
  -> 函数
  -> 异常和文件
  -> 模块和包
  -> 面向对象
  -> 标准库
  -> 第三方库
  -> 项目结构、测试、打包、发布
```

这条路线的核心思想是：先会写小段程序，再会封装逻辑，然后会组织项目，最后会利用生态解决真实问题。

## 2. 环境准备：安装、运行、虚拟环境

### 2.1 安装 Python

可以从 Python 官网下载对应系统的安装包：

- 官网：https://www.python.org/
- 下载页：https://www.python.org/downloads/

Windows 安装时建议勾选：

- `Add Python to PATH`
- `pip`
- `py launcher`

安装完成后，在命令行中检查：

```powershell
python --version
pip --version
```

如果 Windows 上 `python` 命令不可用，可以尝试：

```powershell
py --version
py -m pip --version
```

### 2.2 运行 Python 的几种方式

**方式一：交互式解释器**

适合快速试验语法：

```powershell
python
```

进入后可以输入：

```python
>>> 1 + 2
3
>>> print("Hello Python")
Hello Python
```

退出：

```python
exit()
```

**方式二：运行 `.py` 文件**

创建 `hello.py`：

```python
print("Hello Python")
```

运行：

```powershell
python hello.py
```

**方式三：使用 IDE 或编辑器**

常见选择：

- VS Code：轻量、插件丰富。
- PyCharm：Python 项目管理体验完整。
- Jupyter Notebook：适合数据分析、机器学习和交互式笔记。

### 2.3 虚拟环境：每个项目一套依赖

虚拟环境用于隔离不同项目的依赖。比如项目 A 需要 `requests==2.31`，项目 B 需要更新版本，如果都安装到系统 Python 里，很容易互相影响。

在项目目录中创建虚拟环境：

```powershell
python -m venv .venv
```

Windows PowerShell 激活：

```powershell
.\.venv\Scripts\Activate.ps1
```

Linux/macOS 激活：

```bash
source .venv/bin/activate
```

激活后安装第三方库：

```powershell
python -m pip install requests
```

查看已安装库：

```powershell
python -m pip list
```

导出依赖：

```powershell
python -m pip freeze > requirements.txt
```

从依赖文件安装：

```powershell
python -m pip install -r requirements.txt
```

建议使用 `python -m pip`，这样可以确保调用的是当前 Python 环境对应的 `pip`。

## 3. 基础语法：缩进、注释、标识符与变量

### 3.1 缩进决定代码块

Python 不使用 `{}` 包住代码块，而是使用缩进。

```python
age = 18

if age >= 18:
    print("成年人")
    print("可以继续执行成年逻辑")
else:
    print("未成年人")
```

注意：

- 同一代码块缩进必须一致。
- 推荐使用 4 个空格缩进。
- 不要混用 Tab 和空格。

错误示例：

```python
if True:
    print("第一行")
  print("第二行")  # 缩进不一致，会报错
```

### 3.2 注释

单行注释使用 `#`：

```python
# 计算圆面积
radius = 3
area = 3.14 * radius * radius
```

多行说明通常使用三引号字符串，但严格来说它不是“多行注释”，而是一个字符串字面量。如果它出现在模块、类、函数的开头，就会成为文档字符串 `docstring`。

```python
def add(a, b):
    """返回 a 和 b 的和。"""
    return a + b
```

临时屏蔽多行代码时，更推荐编辑器的批量注释快捷键，而不是乱用三引号。

### 3.3 标识符命名规则

变量名、函数名、类名、模块名都属于标识符。

命名规则：

- 可以由字母、数字、下划线组成。
- 不能以数字开头。
- 区分大小写：`name` 和 `Name` 是两个不同名字。
- 不能使用关键字，如 `if`、`for`、`class`、`return`。
- Python 支持中文变量名，但项目中不推荐这样写。

推荐风格：

```python
user_name = "Alice"      # 变量、函数：小写 + 下划线
MAX_RETRY = 3            # 常量：大写 + 下划线

class UserProfile:       # 类名：大驼峰
    pass
```

查看 Python 关键字：

```python
import keyword

print(keyword.kwlist)
```

### 3.4 变量和赋值

Python 中变量更像是“名字”，它指向某个对象。

```python
x = 10
x = "hello"
```

这并不表示同一个内存位置先变成整数又变成字符串，而是变量名 `x` 先指向整数对象，后来又指向字符串对象。

可以一次给多个变量赋值：

```python
a, b = 1, 2
a, b = b, a  # 交换变量
```

也可以链式赋值：

```python
x = y = z = 0
```

## 4. 输入输出：print、input 与字符串格式化

### 4.1 print 的基本用法

`print()` 用于输出内容。

```python
print(123)
print("Hello World")
print("你好，Python")
```

`print()` 可以一次输出多个对象：

```python
name = "Alice"
age = 18

print(name, age)
```

默认情况下，多个对象之间用空格分隔，最后自动换行。

### 4.2 print 的常用参数

`print()` 的常见形式：

```python
print(*objects, sep=" ", end="\n", file=None, flush=False)
```

常用参数：

- `objects`：要输出的一个或多个对象。
- `sep`：多个对象之间的分隔符，默认是空格。
- `end`：输出结尾，默认是换行 `\n`。
- `file`：输出到哪个文件对象，默认是标准输出。
- `flush`：是否强制刷新缓冲区。

示例：

```python
tmp1 = 1
tmp2 = 2

print(tmp1, tmp2)              # 1 2
print(tmp1, tmp2, sep=",")     # 1,2
print(tmp1, tmp2, sep="-")     # 1-2
print(tmp1, end=" ")
print(tmp2)                    # 1 2
```

输出到文件：

```python
with open("output.txt", "w", encoding="utf-8") as f:
    print("写入文件", file=f)
```

### 4.3 input 的使用

`input()` 从标准输入读取内容，返回值永远是字符串。

```python
name = input("请输入你的名字：")
age = input("请输入你的年龄：")

print(f"你好，{name}！你今年 {age} 岁。")
print(type(age))  # <class 'str'>
```

如果需要数字，必须手动转换：

```python
age = int(input("请输入你的年龄："))

if age >= 18:
    print("成年人")
else:
    print("未成年人")
```

如果用户输入的不是合法数字，`int()` 会报错。后面异常处理部分会讲如何处理。

### 4.4 字符串格式化

Python 中常见格式化方式有三种。

**方式一：`%` 占位符**

```python
name = "Alice"
age = 18
height = 1.68

print("我是 %s，今年 %d 岁，身高 %.2f 米" % (name, age, height))
```

**方式二：`str.format()`**

```python
print("我是 {}，今年 {} 岁，身高 {:.2f} 米".format(name, age, height))
```

**方式三：f-string**

这是最推荐的日常写法：

```python
print(f"我是 {name}，今年 {age} 岁，身高 {height:.2f} 米")
```

f-string 可以直接写表达式：

```python
price = 19.9
count = 3

print(f"总价：{price * count:.2f} 元")
```

常见格式控制：

```python
value = 3.1415926

print(f"{value:.2f}")      # 3.14，保留两位小数
print(f"{value:10.2f}")    # 宽度 10，右对齐
print(f"{value:<10.2f}")   # 宽度 10，左对齐
print(f"{value:^10.2f}")   # 宽度 10，居中
```

## 5. 数据类型：数字、字符串、布尔值与空值

### 5.1 动态类型与强类型

Python 是动态类型语言：

```python
x = 100
x = "hello"
```

变量 `x` 可以在不同时间指向不同类型的对象。

Python 也是强类型语言：

```python
print("1" + 2)  # TypeError
```

字符串和整数不能直接相加，需要明确转换：

```python
print(int("1") + 2)   # 3
print("1" + str(2))   # "12"
```

### 5.2 数字类型

常见数字类型：

- `int`：整数，支持任意精度。
- `float`：浮点数，通常是双精度浮点。
- `complex`：复数。

示例：

```python
a = 1
b = 2.0
c = 1 + 2j

print(type(a))  # <class 'int'>
print(type(b))  # <class 'float'>
print(type(c))  # <class 'complex'>
```

整数没有固定范围限制，只受内存限制：

```python
big = 10 ** 100
print(big)
```

浮点数要注意精度误差：

```python
print(0.1 + 0.2)  # 0.30000000000000004
```

如果涉及金额等精确小数，可以使用 `decimal.Decimal`：

```python
from decimal import Decimal

price = Decimal("0.1") + Decimal("0.2")
print(price)  # 0.3
```

### 5.3 字符串 str

字符串可以用单引号、双引号或三引号：

```python
s1 = "hello"
s2 = 'hello'
s3 = """多行
字符串"""
```

常见操作：

```python
text = "Python"

print(text[0])       # P
print(text[-1])      # n
print(text[0:3])     # Pyt
print(text[::2])     # Pto
print(len(text))     # 6
```

字符串是不可变对象：

```python
text = "Python"
# text[0] = "J"  # TypeError
text = "J" + text[1:]
```

常用方法：

```python
msg = "  hello python  "

print(msg.strip())              # 去掉两侧空白
print(msg.upper())              # 转大写
print(msg.replace("python", "world"))
print("a,b,c".split(","))       # ['a', 'b', 'c']
print("-".join(["a", "b", "c"])) # a-b-c
```

### 5.4 布尔值 bool

布尔值只有两个：

```python
flag1 = True
flag2 = False
```

注意首字母必须大写，不能写成 `true` 或 `false`。

常见假值：

```python
False
None
0
0.0
""
[]
()
{}
set()
```

其他大多数对象都被视为真。

### 5.5 空值 None

`None` 表示“没有值”或“空”。

```python
result = None

if result is None:
    print("还没有结果")
```

判断是否为 `None` 时推荐使用 `is None`，不要使用 `== None`。

### 5.6 类型转换

常见转换函数：

```python
num = 100.5

print(int(num))      # 100，直接截断小数部分
print(float("3.14"))
print(str(123))
print(bool(""))
print(bool("hello"))
```

注意：

- `str()` 几乎可以把任何对象转换成字符串。
- `int("123")` 可以成功，`int("abc")` 会失败。
- `int(3.9)` 是 `3`，不是四舍五入。
- 四舍五入可以用 `round()`。

```python
print(round(3.14159, 2))  # 3.14
```

## 6. 运算符与表达式

### 6.1 算术运算符

```python
a = 9
b = 2

print(a + b)   # 11
print(a - b)   # 7
print(a * b)   # 18
print(a / b)   # 4.5，普通除法，结果通常是 float
print(a // b)  # 4，整除
print(a % b)   # 1，取余
print(a ** b)  # 81，幂运算
```

Python 没有 `++` 和 `--`：

```python
i = 0
i += 1
i -= 1
```

### 6.2 比较运算符

```python
print(3 > 2)
print(3 >= 2)
print(3 < 2)
print(3 <= 2)
print(3 == 2)
print(3 != 2)
```

Python 支持链式比较：

```python
age = 20
print(18 <= age <= 60)
```

### 6.3 逻辑运算符

```python
age = 20
has_ticket = True

print(age >= 18 and has_ticket)
print(age < 18 or has_ticket)
print(not has_ticket)
```

`and` 和 `or` 具有短路特性：

```python
name = ""
display_name = name or "匿名用户"
print(display_name)
```

### 6.4 成员运算符

```python
text = "python"

print("py" in text)
print("java" not in text)
```

也常用于列表、字典、集合：

```python
users = ["Alice", "Bob"]
print("Alice" in users)

scores = {"Alice": 90, "Bob": 80}
print("Alice" in scores)  # 判断键是否存在
```

### 6.5 身份运算符

`is` 判断两个变量是否指向同一个对象。

```python
a = [1, 2]
b = a
c = [1, 2]

print(a == c)  # True，值相等
print(a is c)  # False，不是同一个对象
print(a is b)  # True，是同一个对象
```

日常规则：

- 判断值是否相等，用 `==`。
- 判断是否为 `None`，用 `is None`。

### 6.6 条件表达式

Python 没有 C/Java 那种 `条件 ? A : B`，但有自己的条件表达式：

```python
age = 18
status = "成年" if age >= 18 else "未成年"
print(status)
```

语法顺序：

```text
满足条件时的结果 if 条件 else 不满足条件时的结果
```

## 7. 程序结构：分支、循环与模式匹配

### 7.1 if / elif / else

```python
score = 85

if score >= 90:
    level = "优秀"
elif score >= 80:
    level = "良好"
elif score >= 60:
    level = "及格"
else:
    level = "不及格"

print(level)
```

`elif` 是 `else if` 的缩写。多个分支只会执行第一个满足条件的分支。

### 7.2 match / case

`match-case` 是 Python 3.10 引入的结构化模式匹配，适合处理多种固定结构。

简单值匹配：

```python
command = "start"

match command:
    case "start":
        print("启动")
    case "stop":
        print("停止")
    case "restart":
        print("重启")
    case _:
        print("未知命令")
```

匹配列表结构：

```python
point = [10, 20]

match point:
    case [0, 0]:
        print("原点")
    case [x, 0]:
        print(f"x 轴上的点：{x}")
    case [x, y]:
        print(f"普通点：{x}, {y}")
```

`case _` 相当于默认分支。

### 7.3 while 循环

```python
i = 0

while i < 5:
    print(i)
    i += 1
```

适合“不确定循环次数，但知道结束条件”的场景。

### 7.4 for 循环和 range

`for` 用于遍历可迭代对象。

```python
for i in range(5):
    print(i)  # 0 1 2 3 4
```

`range()` 常见用法：

```python
range(5)        # 0, 1, 2, 3, 4
range(1, 6)     # 1, 2, 3, 4, 5
range(2, 11, 2) # 2, 4, 6, 8, 10
```

遍历字符串：

```python
for ch in "Python":
    print(ch)
```

遍历列表：

```python
names = ["Alice", "Bob", "Charlie"]

for name in names:
    print(name)
```

### 7.5 break、continue 和循环 else

`break` 结束整个循环：

```python
for i in range(10):
    if i == 5:
        break
    print(i)
```

`continue` 跳过本次循环，进入下一次：

```python
for i in range(10):
    if i % 2 == 0:
        continue
    print(i)
```

循环也可以带 `else`。当循环没有被 `break` 打断时，`else` 会执行：

```python
target = 7

for i in range(10):
    if i == target:
        print("找到了")
        break
else:
    print("没找到")
```

## 8. 容器类型：list、tuple、dict、set

### 8.1 list 列表

列表是有序、可变、允许重复元素的容器。

```python
numbers = [1, 2, 3]
names = ["Alice", "Bob"]
mixed = [1, "hello", True]
```

常见操作：

```python
nums = [10, 20, 30]

print(nums[0])
print(nums[-1])
print(nums[0:2])

nums.append(40)
nums.insert(1, 15)
nums.remove(20)
last = nums.pop()

print(nums)
print(last)
```

排序：

```python
nums = [3, 1, 2]

nums.sort()
print(nums)  # [1, 2, 3]

nums.sort(reverse=True)
print(nums)  # [3, 2, 1]
```

`sorted()` 返回新列表，不修改原列表：

```python
nums = [3, 1, 2]
new_nums = sorted(nums)

print(nums)      # [3, 1, 2]
print(new_nums)  # [1, 2, 3]
```

### 8.2 tuple 元组

元组是有序、不可变、允许重复元素的容器。

```python
point = (10, 20)
print(point[0])
```

只有一个元素的元组必须加逗号：

```python
a = (1)
b = (1,)

print(type(a))  # <class 'int'>
print(type(b))  # <class 'tuple'>
```

元组适合表示固定结构的数据，例如坐标、日期片段、函数多个返回值。

```python
def get_user():
    return "Alice", 18

name, age = get_user()
```

### 8.3 dict 字典

字典是键值对容器，适合通过“键”快速查找“值”。

```python
user = {
    "name": "Alice",
    "age": 18,
    "city": "Shanghai",
}

print(user["name"])
```

访问不存在的键会报错：

```python
# print(user["email"])  # KeyError
```

可以使用 `get()`：

```python
print(user.get("email"))
print(user.get("email", "未填写"))
```

增删改：

```python
user["age"] = 19
user["email"] = "alice@example.com"
del user["city"]
```

遍历：

```python
for key in user:
    print(key, user[key])

for key, value in user.items():
    print(key, value)
```

### 8.4 set 集合

集合是无序、不重复的容器，适合去重和集合运算。

```python
nums = {1, 2, 2, 3}
print(nums)  # {1, 2, 3}
```

常见操作：

```python
a = {1, 2, 3}
b = {3, 4, 5}

print(a | b)  # 并集 {1, 2, 3, 4, 5}
print(a & b)  # 交集 {3}
print(a - b)  # 差集 {1, 2}
print(a ^ b)  # 对称差集 {1, 2, 4, 5}
```

创建空集合要用 `set()`：

```python
empty_dict = {}
empty_set = set()
```

### 8.5 可变对象与不可变对象

不可变对象：

- `int`
- `float`
- `str`
- `tuple`
- `bool`
- `None`

可变对象：

- `list`
- `dict`
- `set`

可变对象作为函数默认参数时要特别小心：

```python
def add_item(item, items=[]):  # 不推荐
    items.append(item)
    return items

print(add_item("a"))  # ['a']
print(add_item("b"))  # ['a', 'b']，可能不是你想要的
```

推荐写法：

```python
def add_item(item, items=None):
    if items is None:
        items = []
    items.append(item)
    return items
```

### 8.6 切片

切片语法：

```python
sequence[start:stop:step]
```

特点：左闭右开。

```python
nums = [0, 1, 2, 3, 4, 5]

print(nums[1:4])   # [1, 2, 3]
print(nums[:3])    # [0, 1, 2]
print(nums[3:])    # [3, 4, 5]
print(nums[::2])   # [0, 2, 4]
print(nums[::-1])  # [5, 4, 3, 2, 1, 0]
```

### 8.7 推导式入门

列表推导式：

```python
squares = [x * x for x in range(1, 6)]
print(squares)  # [1, 4, 9, 16, 25]
```

带条件：

```python
even_squares = [x * x for x in range(1, 11) if x % 2 == 0]
```

字典推导式：

```python
score_map = {name: 0 for name in ["Alice", "Bob", "Charlie"]}
```

集合推导式：

```python
unique_lengths = {len(word) for word in ["hi", "hello", "python"]}
```

推导式适合写简短转换，逻辑复杂时应改用普通循环，提高可读性。

## 9. 函数：把重复逻辑封装起来

### 9.1 定义和调用函数

```python
def greet(name):
    print(f"你好，{name}")

greet("Alice")
```

函数可以有返回值：

```python
def add(a, b):
    return a + b

result = add(1, 2)
print(result)
```

没有显式 `return` 时，函数默认返回 `None`。

### 9.2 参数类型

**位置参数**

```python
def power(x, n):
    return x ** n

print(power(2, 3))
```

**默认参数**

```python
def greet(name, prefix="你好"):
    print(f"{prefix}，{name}")

greet("Alice")
greet("Bob", "早上好")
```

**关键字参数**

```python
greet(name="Alice", prefix="晚上好")
```

**可变位置参数**

```python
def total(*numbers):
    return sum(numbers)

print(total(1, 2, 3))
```

**可变关键字参数**

```python
def show_info(**info):
    for key, value in info.items():
        print(key, value)

show_info(name="Alice", age=18)
```

### 9.3 参数解包

```python
def add(a, b):
    return a + b

nums = [1, 2]
print(add(*nums))

params = {"a": 3, "b": 4}
print(add(**params))
```

### 9.4 作用域

```python
x = 10

def func():
    x = 20
    print("函数内部：", x)

func()
print("函数外部：", x)
```

如果需要在函数内部修改全局变量，可以使用 `global`，但实际项目中应尽量减少这种写法。

```python
count = 0

def increase():
    global count
    count += 1
```

更推荐让函数通过参数和返回值交换数据：

```python
def increase(count):
    return count + 1

count = increase(count)
```

### 9.5 文档字符串和类型提示

文档字符串让别人知道函数做什么：

```python
def divide(a, b):
    """返回 a / b 的结果。"""
    return a / b
```

类型提示让代码更易读，也方便 IDE、类型检查工具发现问题：

```python
def add(a: int, b: int) -> int:
    return a + b
```

类型提示不会在运行时自动强制检查：

```python
print(add("1", "2"))  # 仍然可以运行，结果是 "12"
```

如果需要静态检查，可以使用 `mypy`、`pyright` 等工具。

### 9.6 lambda 表达式

`lambda` 用于定义简短匿名函数。

```python
square = lambda x: x * x
print(square(3))
```

更常见于排序：

```python
users = [
    {"name": "Alice", "score": 90},
    {"name": "Bob", "score": 80},
]

users.sort(key=lambda user: user["score"], reverse=True)
print(users)
```

如果逻辑超过一行，应该使用 `def` 定义普通函数。

## 10. 异常处理：让程序面对错误仍然可控

### 10.1 什么是异常

异常是程序运行时出现的问题，例如：

```python
int("abc")      # ValueError
1 / 0           # ZeroDivisionError
{}["name"]      # KeyError
```

如果异常没有被处理，程序会终止并打印错误堆栈。

### 10.2 try / except

```python
try:
    age = int(input("请输入年龄："))
    print(f"明年你 {age + 1} 岁")
except ValueError:
    print("请输入合法数字")
```

捕获多个异常：

```python
try:
    a = int(input("a = "))
    b = int(input("b = "))
    print(a / b)
except ValueError:
    print("请输入数字")
except ZeroDivisionError:
    print("除数不能为 0")
```

### 10.3 else 和 finally

```python
try:
    num = int(input("请输入数字："))
except ValueError:
    print("输入错误")
else:
    print("转换成功：", num)
finally:
    print("无论是否出错都会执行")
```

含义：

- `try`：尝试执行可能出错的代码。
- `except`：处理异常。
- `else`：没有异常时执行。
- `finally`：无论是否异常都会执行，常用于释放资源。

### 10.4 主动抛出异常

```python
def set_age(age):
    if age < 0:
        raise ValueError("年龄不能为负数")
    return age
```

不要滥用异常。对于普通条件判断，用 `if` 就够；对于违反函数约定、数据不合法等情况，可以抛异常。

## 11. 文件读写与上下文管理器

### 11.1 open 的基本用法

```python
file = open("data.txt", "r", encoding="utf-8")
content = file.read()
file.close()
```

这种写法的问题是：如果中途报错，`close()` 可能不会执行。

更推荐使用 `with`：

```python
with open("data.txt", "r", encoding="utf-8") as f:
    content = f.read()
```

`with` 会在代码块结束后自动关闭文件。

### 11.2 文件模式

常见模式：

- `r`：读取，文件不存在会报错。
- `w`：写入，文件不存在会创建，文件存在会清空。
- `a`：追加，写到文件末尾。
- `x`：创建新文件，文件已存在会报错。
- `b`：二进制模式，如 `rb`、`wb`。
- `+`：读写模式，如 `r+`。

写入文件：

```python
with open("notes.txt", "w", encoding="utf-8") as f:
    f.write("第一行\n")
    f.write("第二行\n")
```

追加文件：

```python
with open("notes.txt", "a", encoding="utf-8") as f:
    f.write("追加一行\n")
```

逐行读取：

```python
with open("notes.txt", "r", encoding="utf-8") as f:
    for line in f:
        print(line.strip())
```

### 11.3 JSON 文件

JSON 是项目中常见的数据交换格式。

```python
import json

user = {
    "name": "Alice",
    "age": 18,
    "skills": ["Python", "SQL"],
}

with open("user.json", "w", encoding="utf-8") as f:
    json.dump(user, f, ensure_ascii=False, indent=2)
```

读取：

```python
with open("user.json", "r", encoding="utf-8") as f:
    data = json.load(f)

print(data["name"])
```

`ensure_ascii=False` 可以让中文正常保存，而不是被转义成 `\uXXXX`。

### 11.4 pathlib：更现代的路径处理

```python
from pathlib import Path

path = Path("data") / "users.txt"
path.parent.mkdir(parents=True, exist_ok=True)
path.write_text("hello", encoding="utf-8")

content = path.read_text(encoding="utf-8")
print(content)
```

相比手动拼接字符串，`pathlib.Path` 更清晰，也更容易跨平台。

## 12. 模块、包与导入机制

### 12.1 什么是模块

一个 `.py` 文件就是一个模块。

例如 `math_tools.py`：

```python
def add(a, b):
    return a + b
```

在另一个文件中使用：

```python
import math_tools

print(math_tools.add(1, 2))
```

### 12.2 import 的几种方式

导入整个模块：

```python
import math

print(math.sqrt(16))
```

导入模块中的某个对象：

```python
from math import sqrt

print(sqrt(16))
```

起别名：

```python
import pandas as pd
```

不推荐：

```python
from math import *
```

因为这会把很多名字直接放进当前命名空间，容易造成命名冲突，也不利于阅读。

### 12.3 什么是包

包是包含多个模块的文件夹。传统上，包目录中会有 `__init__.py` 文件。

```text
my_project/
  app/
    __init__.py
    main.py
    utils.py
```

导入：

```python
from app.utils import some_function
```

### 12.4 `if __name__ == "__main__"`

当一个 Python 文件被直接运行时，`__name__` 的值是 `"__main__"`；当它被导入时，`__name__` 是模块名。

```python
def main():
    print("程序入口")

if __name__ == "__main__":
    main()
```

好处：

- 文件可以直接运行。
- 文件也可以被其他模块导入，而不会自动执行主流程。

## 13. 面向对象编程：类、对象、继承与组合

### 13.1 为什么需要面向对象

当程序规模变大后，只有变量和函数会变得混乱。面向对象的核心思想是把“数据”和“操作数据的方法”组织到一起。

例如，一个学生有姓名、年龄、成绩，也有自我介绍、判断是否及格等行为。

### 13.2 定义类和创建对象

```python
class Student:
    def __init__(self, name, age, score):
        self.name = name
        self.age = age
        self.score = score

    def introduce(self):
        print(f"我叫 {self.name}，今年 {self.age} 岁")

    def is_passed(self):
        return self.score >= 60


student = Student("Alice", 18, 90)
student.introduce()
print(student.is_passed())
```

解释：

- `class Student` 定义一个类。
- `__init__` 是初始化方法，创建对象时自动调用。
- `self` 表示当前对象本身。
- `self.name` 是对象属性。
- `introduce()`、`is_passed()` 是对象方法。

### 13.3 实例属性和类属性

实例属性属于某个具体对象：

```python
class User:
    def __init__(self, name):
        self.name = name

u1 = User("Alice")
u2 = User("Bob")
```

`u1.name` 和 `u2.name` 互不影响。

类属性属于类本身，所有实例共享：

```python
class User:
    count = 0

    def __init__(self, name):
        self.name = name
        User.count += 1


u1 = User("Alice")
u2 = User("Bob")
print(User.count)  # 2
```

### 13.4 封装和 property

Python 没有强制私有属性，但约定以下划线表示“内部使用”。

```python
class BankAccount:
    def __init__(self, owner, balance):
        self.owner = owner
        self._balance = balance

    @property
    def balance(self):
        return self._balance

    def deposit(self, amount):
        if amount <= 0:
            raise ValueError("存款金额必须大于 0")
        self._balance += amount

    def withdraw(self, amount):
        if amount <= 0:
            raise ValueError("取款金额必须大于 0")
        if amount > self._balance:
            raise ValueError("余额不足")
        self._balance -= amount
```

使用：

```python
account = BankAccount("Alice", 100)
account.deposit(50)
print(account.balance)
```

`@property` 可以让方法像属性一样访问，同时保留内部控制逻辑。

### 13.5 继承

继承用于表达“某个类是一种更具体的类”。

```python
class Animal:
    def __init__(self, name):
        self.name = name

    def speak(self):
        print("动物发出声音")


class Dog(Animal):
    def speak(self):
        print(f"{self.name}：汪")


dog = Dog("小黑")
dog.speak()
```

子类可以复用父类属性和方法，也可以重写父类方法。

使用 `super()` 调用父类方法：

```python
class Employee:
    def __init__(self, name, salary):
        self.name = name
        self.salary = salary


class Manager(Employee):
    def __init__(self, name, salary, department):
        super().__init__(name, salary)
        self.department = department
```

### 13.6 组合优先于继承

不是所有“复用代码”的场景都适合继承。很多时候组合更清晰。

```python
class Engine:
    def start(self):
        print("发动机启动")


class Car:
    def __init__(self):
        self.engine = Engine()

    def start(self):
        self.engine.start()
        print("汽车启动")
```

继承表达“是一个”，组合表达“有一个”。

### 13.7 魔术方法

魔术方法是前后都有双下划线的方法，例如 `__init__`、`__str__`。

```python
class Book:
    def __init__(self, title, author):
        self.title = title
        self.author = author

    def __str__(self):
        return f"《{self.title}》 - {self.author}"


book = Book("Python 入门", "Alice")
print(book)
```

常见魔术方法：

- `__init__`：初始化对象。
- `__str__`：给用户看的字符串。
- `__repr__`：给开发者看的字符串。
- `__len__`：支持 `len(obj)`。
- `__eq__`：支持 `obj1 == obj2`。
- `__lt__`：支持排序比较。

### 13.8 dataclass：减少样板代码

如果一个类主要用于保存数据，可以使用 `dataclasses.dataclass`。

```python
from dataclasses import dataclass


@dataclass
class Student:
    name: str
    age: int
    score: float


student = Student("Alice", 18, 90.5)
print(student)
```

`dataclass` 会自动生成 `__init__`、`__repr__`、`__eq__` 等常用方法。

## 14. 进阶语法：推导式、迭代器、生成器、装饰器

### 14.1 enumerate 和 zip

遍历时同时拿到下标：

```python
names = ["Alice", "Bob", "Charlie"]

for index, name in enumerate(names, start=1):
    print(index, name)
```

同时遍历多个序列：

```python
names = ["Alice", "Bob"]
scores = [90, 80]

for name, score in zip(names, scores):
    print(name, score)
```

### 14.2 迭代器

能被 `for` 遍历的对象叫可迭代对象，例如列表、字符串、字典、文件对象。

```python
nums = [1, 2, 3]
iterator = iter(nums)

print(next(iterator))
print(next(iterator))
print(next(iterator))
```

`for` 循环本质上就是不断调用 `next()`，直到遇到 `StopIteration`。

### 14.3 生成器

生成器用 `yield` 逐个产生值，而不是一次性把所有结果放进内存。

```python
def count_up_to(n):
    i = 1
    while i <= n:
        yield i
        i += 1


for num in count_up_to(3):
    print(num)
```

适合处理大量数据、流式数据。

生成器表达式：

```python
squares = (x * x for x in range(10))
```

### 14.4 装饰器

装饰器用于在不修改原函数内部代码的情况下，为函数增加额外能力。

```python
def log_call(func):
    def wrapper(*args, **kwargs):
        print(f"调用函数：{func.__name__}")
        return func(*args, **kwargs)

    return wrapper


@log_call
def add(a, b):
    return a + b


print(add(1, 2))
```

实际项目中，装饰器常用于：

- 日志记录。
- 权限检查。
- 缓存。
- 统计耗时。
- Web 路由注册。

更规范的装饰器会使用 `functools.wraps` 保留原函数元信息：

```python
from functools import wraps


def log_call(func):
    @wraps(func)
    def wrapper(*args, **kwargs):
        print(f"调用函数：{func.__name__}")
        return func(*args, **kwargs)

    return wrapper
```

## 15. 常用标准库路线图

Python 标准库非常丰富。初学阶段不需要全部背下来，但应知道常用模块能解决什么问题。

### 15.1 pathlib：路径处理

```python
from pathlib import Path

root = Path.cwd()
for path in root.glob("*.py"):
    print(path.name)
```

### 15.2 os 和 sys：系统交互

```python
import os
import sys

print(os.environ.get("PATH"))
print(sys.argv)
```

现在处理路径优先考虑 `pathlib`；需要环境变量、进程相关功能时再使用 `os`、`sys`。

### 15.3 datetime：日期时间

```python
from datetime import datetime, timedelta

now = datetime.now()
tomorrow = now + timedelta(days=1)

print(now.strftime("%Y-%m-%d %H:%M:%S"))
print(tomorrow)
```

### 15.4 json 和 csv：数据读写

```python
import csv

rows = [
    {"name": "Alice", "score": 90},
    {"name": "Bob", "score": 80},
]

with open("scores.csv", "w", newline="", encoding="utf-8") as f:
    writer = csv.DictWriter(f, fieldnames=["name", "score"])
    writer.writeheader()
    writer.writerows(rows)
```

### 15.5 re：正则表达式

```python
import re

text = "邮箱：alice@example.com"
match = re.search(r"[\w.-]+@[\w.-]+", text)

if match:
    print(match.group())
```

正则适合处理“有规律的文本”，但不要用正则解析复杂 HTML、JSON 这类有专门解析器的数据。

### 15.6 logging：日志

相比 `print()`，日志更适合项目：

```python
import logging

logging.basicConfig(
    level=logging.INFO,
    format="%(asctime)s [%(levelname)s] %(message)s",
)

logging.info("程序启动")
logging.warning("这是警告")
logging.error("这是错误")
```

### 15.7 argparse：命令行参数

```python
import argparse

parser = argparse.ArgumentParser(description="问候程序")
parser.add_argument("name", help="用户名")
parser.add_argument("--times", type=int, default=1, help="输出次数")

args = parser.parse_args()

for _ in range(args.times):
    print(f"Hello, {args.name}")
```

运行：

```powershell
python greet.py Alice --times 3
```

### 15.8 unittest：内置测试框架

```python
import unittest


def add(a, b):
    return a + b


class TestAdd(unittest.TestCase):
    def test_add(self):
        self.assertEqual(add(1, 2), 3)


if __name__ == "__main__":
    unittest.main()
```

实际项目中也常用第三方测试框架 `pytest`。

### 15.9 sqlite3：轻量数据库

```python
import sqlite3

with sqlite3.connect("app.db") as conn:
    cursor = conn.cursor()
    cursor.execute("""
        CREATE TABLE IF NOT EXISTS users (
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            name TEXT NOT NULL
        )
    """)
    cursor.execute("INSERT INTO users (name) VALUES (?)", ("Alice",))
    conn.commit()
```

`sqlite3` 适合小项目、本地工具、课程作业和原型系统。

## 16. 第三方库与生态：学会站在工具之上

### 16.1 pip 和 PyPI

PyPI 是 Python 第三方包的主要发布平台，`pip` 是常用包安装工具。

安装包：

```powershell
python -m pip install requests
```

指定版本：

```powershell
python -m pip install requests==2.32.3
```

升级包：

```powershell
python -m pip install --upgrade requests
```

卸载包：

```powershell
python -m pip uninstall requests
```

### 16.2 常见第三方库按方向整理

**网络请求**

- `requests`：简单易用的 HTTP 请求库。
- `httpx`：支持同步和异步的 HTTP 客户端。

示例：

```python
import requests

response = requests.get("https://httpbin.org/get", timeout=10)
print(response.status_code)
print(response.json())
```

**网页解析**

- `beautifulsoup4`：解析 HTML/XML。
- `lxml`：高性能 XML/HTML 解析。

**数据分析**

- `numpy`：数值计算。
- `pandas`：表格数据处理。
- `matplotlib`：绘图。
- `seaborn`：统计图表。

示例：

```python
import pandas as pd

df = pd.read_csv("scores.csv")
print(df.head())
print(df["score"].mean())
```

**Web 开发**

- `Flask`：轻量 Web 框架。
- `FastAPI`：现代 API 框架，类型提示支持好，自动生成接口文档。
- `Django`：功能完整的 Web 框架，适合中大型网站。

**测试和代码质量**

- `pytest`：更简洁强大的测试框架。
- `ruff`：快速的代码检查和格式化工具。
- `black`：代码格式化。
- `mypy`：静态类型检查。

**自动化和文件处理**

- `openpyxl`：读写 Excel。
- `python-docx`：读写 Word。
- `Pillow`：图像处理。
- `playwright`：浏览器自动化。

**数据库和后端**

- `SQLAlchemy`：数据库 ORM 和 SQL 工具。
- `psycopg`：PostgreSQL 驱动。
- `pymysql`：MySQL 驱动。

### 16.3 选择第三方库的判断标准

选择库时不要只看“能不能用”，还要看：

- 是否仍在维护。
- 文档是否清晰。
- 下载量和社区使用情况。
- 是否支持当前 Python 版本。
- 许可证是否适合项目。
- 是否过度复杂。

初学阶段建议先用主流库，少追求冷门替代品。

## 17. 项目实战管理：从脚本到可维护项目

### 17.1 一个推荐的项目结构

```text
my_project/
  .venv/
  src/
    my_project/
      __init__.py
      main.py
      services.py
  tests/
    test_services.py
  README.md
  pyproject.toml
  requirements.txt
  .gitignore
```

解释：

- `.venv/`：虚拟环境，不提交到 Git。
- `src/my_project/`：项目源代码。
- `tests/`：测试代码。
- `README.md`：项目说明。
- `pyproject.toml`：现代 Python 项目配置文件。
- `requirements.txt`：简单项目可用的依赖锁定文件。
- `.gitignore`：忽略缓存、虚拟环境等文件。

### 17.2 pyproject.toml

`pyproject.toml` 是现代 Python 项目的核心配置文件，可以配置构建系统、项目元数据、格式化工具、测试工具等。

简单示例：

```toml
[project]
name = "my-project"
version = "0.1.0"
description = "A simple Python project"
requires-python = ">=3.12"
dependencies = [
  "requests>=2.32",
]

[project.scripts]
my-project = "my_project.main:main"

[tool.pytest.ini_options]
testpaths = ["tests"]

[tool.ruff]
line-length = 88
```

如果只是课程作业或简单脚本，不一定马上需要完整打包；但如果项目会长期维护，建议尽早使用清晰的结构。

### 17.3 requirements.txt

简单项目可以用：

```text
requests==2.32.3
pandas==2.2.3
```

安装：

```powershell
python -m pip install -r requirements.txt
```

`requirements.txt` 适合记录运行环境依赖。更复杂的项目可以使用 `pyproject.toml`、`uv`、`poetry`、`pdm` 等工具管理依赖。

### 17.4 测试

安装 `pytest`：

```powershell
python -m pip install pytest
```

示例 `tests/test_math_tools.py`：

```python
from my_project.services import add


def test_add():
    assert add(1, 2) == 3
```

运行：

```powershell
python -m pytest
```

测试的价值：

- 防止修改代码后破坏旧功能。
- 帮助你明确函数输入输出。
- 项目越大越重要。

### 17.5 代码风格与质量

安装 `ruff`：

```powershell
python -m pip install ruff
```

检查：

```powershell
python -m ruff check .
```

格式化：

```powershell
python -m ruff format .
```

代码质量不只是“好看”，它能降低后期维护成本。

### 17.6 Git 基础协作习惯

常用 `.gitignore`：

```gitignore
.venv/
__pycache__/
*.pyc
.pytest_cache/
.ruff_cache/
.mypy_cache/
.env
```

基本流程：

```powershell
git status
git add .
git commit -m "Add todo cli"
```

提交信息建议写清楚“做了什么”，不要只写 `update`。

### 17.7 配置和密钥

不要把密码、Token、数据库连接串写死在代码中。

可以通过环境变量读取：

```python
import os

api_key = os.environ.get("API_KEY")
if not api_key:
    raise RuntimeError("请设置 API_KEY 环境变量")
```

本地开发常见做法是使用 `.env` 文件，并把 `.env` 加入 `.gitignore`。

## 18. 小项目示例：命令行待办清单

这个小项目把前面学到的内容串起来：函数、文件、JSON、命令行参数、列表、字典和异常处理。

### 18.1 功能目标

实现一个 `todo.py`：

- 添加任务。
- 查看任务列表。
- 标记任务完成。
- 删除任务。
- 数据保存到本地 `todos.json`。

### 18.2 完整代码

```python
import argparse
import json
from pathlib import Path


DATA_FILE = Path("todos.json")


def load_todos():
    if not DATA_FILE.exists():
        return []

    with DATA_FILE.open("r", encoding="utf-8") as f:
        return json.load(f)


def save_todos(todos):
    with DATA_FILE.open("w", encoding="utf-8") as f:
        json.dump(todos, f, ensure_ascii=False, indent=2)


def add_todo(title):
    todos = load_todos()
    todo = {
        "id": next_id(todos),
        "title": title,
        "done": False,
    }
    todos.append(todo)
    save_todos(todos)
    print(f"已添加：{title}")


def list_todos():
    todos = load_todos()
    if not todos:
        print("暂无任务")
        return

    for todo in todos:
        mark = "x" if todo["done"] else " "
        print(f'{todo["id"]:>3}. [{mark}] {todo["title"]}')


def finish_todo(todo_id):
    todos = load_todos()
    for todo in todos:
        if todo["id"] == todo_id:
            todo["done"] = True
            save_todos(todos)
            print(f"已完成：{todo['title']}")
            return

    print(f"未找到任务：{todo_id}")


def delete_todo(todo_id):
    todos = load_todos()
    new_todos = [todo for todo in todos if todo["id"] != todo_id]

    if len(new_todos) == len(todos):
        print(f"未找到任务：{todo_id}")
        return

    save_todos(new_todos)
    print(f"已删除任务：{todo_id}")


def next_id(todos):
    if not todos:
        return 1
    return max(todo["id"] for todo in todos) + 1


def build_parser():
    parser = argparse.ArgumentParser(description="命令行待办清单")
    subparsers = parser.add_subparsers(dest="command", required=True)

    add_parser = subparsers.add_parser("add", help="添加任务")
    add_parser.add_argument("title", help="任务标题")

    subparsers.add_parser("list", help="查看任务")

    done_parser = subparsers.add_parser("done", help="完成任务")
    done_parser.add_argument("id", type=int, help="任务 ID")

    delete_parser = subparsers.add_parser("delete", help="删除任务")
    delete_parser.add_argument("id", type=int, help="任务 ID")

    return parser


def main():
    parser = build_parser()
    args = parser.parse_args()

    match args.command:
        case "add":
            add_todo(args.title)
        case "list":
            list_todos()
        case "done":
            finish_todo(args.id)
        case "delete":
            delete_todo(args.id)


if __name__ == "__main__":
    main()
```

### 18.3 运行示例

添加任务：

```powershell
python todo.py add "学习 Python 函数"
python todo.py add "整理项目结构"
```

查看任务：

```powershell
python todo.py list
```

完成任务：

```powershell
python todo.py done 1
```

删除任务：

```powershell
python todo.py delete 2
```

### 18.4 这个项目用到了什么

- `argparse`：处理命令行参数。
- `json`：保存和读取结构化数据。
- `pathlib.Path`：处理文件路径。
- 函数拆分：每个函数只做一件事。
- `match-case`：根据命令分发逻辑。
- 列表推导式：删除任务时生成新列表。

### 18.5 可以继续扩展的方向

- 添加任务创建时间。
- 支持任务优先级。
- 支持按完成状态筛选。
- 增加单元测试。
- 改成 SQLite 存储。
- 打包成命令行工具。

## 19. 学习建议与常见误区

### 19.1 不要只背语法，要多写小程序

语法本身并不难，真正重要的是能把语法组合成解决问题的程序。每学完一块内容，都可以写一个小练习：

- 学完循环：写九九乘法表、猜数字。
- 学完列表和字典：写学生成绩管理。
- 学完文件：写通讯录保存读取。
- 学完函数：把前面的程序重构成函数。
- 学完面向对象：写图书、学生、银行账户等类。
- 学完第三方库：写天气查询、Excel 自动处理、简单 Web API。

### 19.2 常见错误整理

`True`、`False`、`None` 首字母必须大写：

```python
flag = True
```

`print()` 参数之间要用逗号隔开：

```python
print(1, 2, sep=",")
```

字符串和数字不能直接相加：

```python
age = 18
print("年龄：" + str(age))
```

列表越界会报错：

```python
nums = [1, 2, 3]
# print(nums[3])  # IndexError
```

字典访问不存在的键会报错：

```python
user = {"name": "Alice"}
print(user.get("age", 0))
```

浮点数不能简单用来表示精确金额：

```python
from decimal import Decimal

money = Decimal("0.1") + Decimal("0.2")
```

### 19.3 什么时候该拆函数、拆文件、写类

可以用这几个判断：

- 同一段逻辑复制了两次以上：考虑拆函数。
- 一个文件超过几百行且包含多个主题：考虑拆模块。
- 数据和操作总是一起出现：考虑写类。
- 类只是为了放几个函数，没有状态：可能用普通函数更合适。
- 继承层级越来越深：考虑组合。

### 19.4 基础总结

1. 基础语法：计算器、成绩等级判断、猜数字。
2. 容器：学生成绩管理、词频统计。
3. 文件：通讯录、待办清单、日志分析。
4. 函数：把旧脚本重构成多个函数。
5. 面向对象：银行账户、图书管理、简单游戏角色。
6. 标准库：命令行工具、JSON 配置读取、CSV 数据处理。
7. 第三方库：请求网页、处理 Excel、绘制图表。
8. 项目管理：虚拟环境、依赖文件、测试、格式化、Git。

真正学会 Python 的标志，不是记住所有语法，而是知道面对一个需求时：

- 数据应该用什么结构保存。
- 逻辑应该拆成哪些函数。
- 哪些错误需要处理。
- 哪些功能可以使用标准库或第三方库。
- 项目如何组织，别人才能看懂和继续维护。

