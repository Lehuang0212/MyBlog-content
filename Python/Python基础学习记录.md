---
title: Python基础学习记录
date: 2025-10-28 22:21:15
tags:
---

本篇文章为我的第一篇正式文章，用于记录我的python基础学习。
<!-- more -->

# 一、Python的基本语法
pyton是一门十分简洁的语言，代码没有C++和Java那样严谨至极，每行代码结束也不再需要以分号(;)结尾。
在进行分支/循环/函数等结构的编写时，也不再需要大括号去严格括住代码结构体，以代码的缩进为主要的判断方式。
Python中标识符一样有着命名规则，与C++和JAVA同样有着：内容限制、大小写敏感、不可使用关键字。
在标识符的命名中只允许出现：英文、中文、数组以及下划线(_)其余任何内容都不被允许，虽然支持中文，但并不推荐使用中文命名，与C++和JAVA相同，Python也不允许变量以数字开头。
在Python中，#代表单行注释，若要使用多行注释要以三个双引号把多行内容引用起来。
例如：
```python
# 这是单行注释

"""
    这是多行注释
    可以进行多行的注释操作
"""
```
## ① input和print(输入和输出)
### 1.1 print的使用
Python的输出使用print函数，其常规使用方法就是print(内容),但是print()函数仍有很多使用方法。
例如：
```python
# 常规用法
print(123456)
print("Hello World!")
print("你好呀！")
print(
    """
        多行格式输出
        111
            222
                333
    """
)
```
print()的函数定义为
```python
print(*objects, sep=' ', end='\n', file=sys.stdout, flush=False)
```
其中：
objects --> 对象，表示可以一次输出多个对象。输出多个对象时，需要用 , 分隔
```python
tmp1 = 1
tmp2 = 2
print(tmp1,tmp2)
# 输出结果是1 2
```
sep --> 用来间隔多个对象，默认值是一个空格。
```python
tmp1 = 1
tmp2 = 2
print(tmp1,tmp2 sep=',')
# 输出结果是1,2
print(tmp1,tmp2 sep='-')
# 输出结果是1-2
```
end --> 用来设定以什么结尾。默认值是换行符 \n，我们可以换成其他字符串。
```python
tmp1 = 1
tmp2 = 2
print(tmp1，end = '第一次输出\n')
print(tmp2,end = '\n先换行,第二行输出')
"""
输出结果：
1第一次输出
2
第二次输出
"""
```
file --> 要写入的文件对象。
flush --> 输出是否被缓存通常决定于 file，但如果 flush 关键字参数为 True，流会被强制刷新。
### 1.2 print格式化输出
Python中进行格式化输出时，有三种方式：占位符%输出、format格式化、f-String格式化。
例如：
```python
# %占位符输出方式
name = "Lehuang"
age = 19
height = 1.68
print("我是%s,我今年%d岁,身高%fm" % (name,age,height))

# format格式化输出
print("我是{}，我今年{}岁,身高{}m".format(name,age,height))

# f-String格式化

print(f"我是{name},我今年{age}岁,身高{height}m")

# 输出 我是Lehuang,我今年19岁,身高1.68m

print(f"我是{name},我今年{age+1}岁,身高{height:<10.2f}m")
# f-String可以进行格式控制，比如age+1，和保留两位小数及宽度为10和进行左对齐等操作
# 对齐方式（<左对齐、>右对齐、^居中）和宽度需要放在精度（.2f）之前，正确的语法顺序是：[对齐][宽度].[精度][类型]。

```
### 1.3 input的使用
input()是 Python 内置的函数，核心作用是从标准输入（通常是键盘）读取用户输入的内容，并将读取到的内容以字符串（str）类型返回。
例如:
```python
# 等待用户输入，无提示语
user_input = input()
# 打印输入内容和类型
print("你输入的内容是：", user_input)
print("内容的类型是：", type(user_input)) # 内容的类型是： <class 'str'> 注意：即使输入数字，类型也是字符串

# 带提示语的输入
name = input("请输入你的名字：")
age = input("请输入你的年龄：")

# 输出结果
print(f"你好，{name}！你今年{age}岁了。")
```
## ②Python的数据类型
### 2.1基本数据类型
python中变量的数据类型一般是自动(弱数据类型)，程序员不需要思考存储的变量的范围合类型，由系统去判定。
例如：
```python
# 变量的初始化及赋值
a = 1
b = 2.0
c = 2.00
d = "这是一个字符串"
# type()语句可以得到数据的类型
print(type(a))
print(type(b))
print(type(c))
print(type(d))
"""
输出结果为：
<class 'int'>
<class 'float'>
<class 'float'>
<class 'str'>
"""
```
Python中的int类型是自带高精度的，无需像C++那样考虑精度问题，也无需像JAVA那样使用BigInteger特殊类来解决高精度问题。
在Python中float代表的是浮点型，并不区分单精度和双精度。

### 2.2基本数据类型的转换

在Python中，常见的类型转换语句为：
```python
num = 100.00
print(type(num))
print(type(int(num)))
print(type(float(num)))
print(type(str(num)))
"""
这段代码的输出是：
<class 'float'>
<class 'int'>
<class 'float'>
<class 'str'>
"""
```

需要注意的是int() float() str()都是带有返回值的，并不是执行完int(num)之后，num存储的数据就永久变为了int类型，即只是在执行int(num)语句时转换为了int类型。
类型转换也不是万能的，对于字符串的转换需要注意的是:1.任何类型，都可以通过str()，转换成字符串。2.字符串内必须真的是数字，才可以将字符串转换为数字。
对于浮点数的转换，当浮点数转换为整数时，会直接舍去小数部分，使用时要注意精度丢失。

### 2.3运算符
运算符方面，Python中大多运算符都是熟知的，与C++和JAVA相同的，但是仍有区分。
**算术运算符**
加减乘除都是没有区分的，使用方式都是相同的，但是整除和指数运算并不相同，在Python中，//代表整除，舍去余数。
\**代表指数运算，比如当a=10，b=20时，a\**b代表10的20次方。
注意:在python中并没有++和--这种自增自减操作，需要对于变量自身进行操作时应使用赋值运算符:+=、-=、*=、/=、//=、%=、**=。
例如：
```python
a = 9
b = 2
# 9/2 = 4.5
print(a/b)
# 9//2 = 4
print(a//b)
# 2的9次方为512
print(b**a)
"""
输出结果为：
4.5
4
512
"""
```
**逻辑运算符**
与为and、或为or，非为not。与运算一假必假、或运算一真必真、非运算假变真，真变假。
**成员运算符**
in为某个数据是否是某个容器的成员，not in为某个数据是否不是某个容器的成员
**身份运算符**
is判断两个数据是否为同一个内存地址
id()获取某个变量地址的整数形式
**三目运算符**
Python中没有三目运算符，但会使用分支结构进行变形，当做三目用
条件满足执行的结果if判断 else条件不满足执行的结果
```python
print("成年"if age >= 18 else "未成年")
```

## ③python程序结构
顺序执行：自上而下依次执行
分支-多分支：选择其中一条执行
循环：按照条件重复执行
### 3.1 分支结构
if分支结构: 
    if-elif-else
match分支结构:
    match(): 
        case 1: 
        case 2: 
        ...
例如：
```python
# if-elif-else嵌套
flag = true
flg = false
if flag:
    print("flag true")
elif not flg:
    print("flg false")
else:
    print("flag false and flg true")

# match-case结构
match num:
    case 1:
        ...
    case 2:
        ...
    case _: #替代default，匹配所有未被前面case覆盖的情况
        ...
```

### 3.2 循环结构
**whiile循环**: while 条件:
**for循环**: for in range()
**range函数**:产生整数序列(左闭右开区间)
    range(x) 产生 0~x-1; 
    range(x,y) 产生x~y-1;
    range(x,y,z)从x开始，每次加z产生一个数，z为步长
```python
# while循环10次
i = 0
while i < 10 :
    i += 1

# for循环10次
for i in range(10):
    print(i,end=",") # 0,1,2,3,4,5,6,7,8,9,
for i in range(1,11):
    print(i,end=",") # 1,2,3,4,5,6,7,8,9,10,
for i in range(2,21,2):
    print(i,end=",") # 2,4,6,8,10,12,14,16,18,20,
```
