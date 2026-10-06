# Python基础阶段考试

（作者：尚硅谷研究院） 

## 一、选择题（10题，每题 3 分，共 30分）

1. 以下哪个是 Python 中的合法变量名？（C ）

A. 2var

B. var-1

C. _var1

D. var@1

2. 以下哪个是 Python 中的合法变量名？（B）

A. 1name

B. name_1

C. name!

D. for

3. 下列数据类型中，不可变的数据类型是（D ）

A. list

B. dict

C. set

D. tuple

4. 下列数据类型中，属于可变数据类型的是（C）

A. str

B. tuple

C. list

D. int

5. 执行以下代码后，result的值是多少？（A ）

a = 5

b = 3

result = a // b

A. 1

B. 1.6666666666666667

C. 2

D. 0

6. 行代码a = 10 % 3后，a 的值为（B）

A. 3

B. 1

C. 0

D. 3.333

7. 以下关于if - elif - else语句的说法，正确的是（A ）

A. elif语句可以有多个，也可以没有

B. else语句必须跟在if语句后面，不能跟在elif语句后面

C. if - elif - else语句中，每个分支都至少会执行一次

D. if语句后面只能跟一个elif语句

8. 以下哪个函数用于将字符串转换为整数？（ B）

A. str()

B. int()

C. float()

D. list()

9. 以下哪个函数用于将整数转换为字符串？（C）

A. int ()

B. float ()

C. str ()

D. list ()

10. 以下关于函数的说法，错误的是（C ）

A. 函数定义时，参数可以有默认值

B. 函数可以返回多个值

C. 函数内部定义的变量是全局变量

D. 函数可以没有参数

11. 关于 Python 函数，说法正确的是（C）

A. 函数必须有返回值

B. 函数参数不能设置默认值

C. 函数可以嵌套定义

D. 函数内部变量默认是全局变量

12. 以下哪个模块用于处理日期和时间？（C ）

A. math

B. random

C. datetime

D. os

13. 执行以下代码后，输出结果是什么？（A ）

```python
nums = [1, 2, 3, 4, 5]

for num in nums:

  if num == 3:

    break

  print(num)

```

A. 1 2

B. 1 2 3

C. 1 2 3 4 5

D. 1 2 4 5

14. 执行以下代码，输出结果是（"p\n"   “y\n"   "t\n" "o\n"  "n\n"）

```python
s = "python"
for i in s:
    if i == "h":
        continue
    print(i)
```



15. 关于 Python 循环语句，以下说法错误的是（D）

A. break 用于跳出当前循环

B. continue 用于跳过本次循环剩余语句

C. while 循环必须设置初始条件

D. for 循环不能遍历字符串

16. 以下关于字典的说法，正确的是（B ）

A. 字典中的键可以重复

B. 字典中的值可以重复

C. 字典是有序的

D. 字典只能通过键来访问值，不能通过索引

17. 关于 Python 列表，说法错误的是（C）

A. 列表可存储不同类型数据

B. 列表支持索引访问

C. 列表长度不可变

D. 列表可使用 append () 添加元素

18. 以下代码定义了一个函数，该函数的功能是（B ）

```python
def sum_list(lst):

  total = 0

  for num in lst:

    total += num

  return total

```

A. 计算列表中所有元素的平均值

B. 计算列表中所有元素的总和

C. 对列表进行排序

D. 查找列表中的最大值

19. 以下代码的功能是（B）

```python
def find_max(lst):
    max_num = lst[0]
    for i in lst:
        if i > max_num:
            max_num = i
    return max_num
```

A. 求列表最小值

B. 求列表最大值

C. 列表求和

D. 列表排序

20. 关于 Python 变量作用域，错误的是（D）

A. 函数内变量默认是局部作用域

B. global 可将局部变量声明为全局

C. nonlocal 用于修改外层函数变量

D. 局部变量可在函数外直接访问

21. 关于装饰器，说法正确的是（B）

A. 装饰器不能带参数

B. 装饰器用于在不修改原函数代码的情况下增强功能

C. 装饰器只能装饰一次

D. 装饰器必须返回 None

22. 关于生成器（generator），错误的是（C）

A. 使用 yield 关键字

B. 节省内存，惰性计算

C. 生成器只能遍历一次

D. 生成器和列表完全一样

23. 关于迭代器（iterator），正确的是（B）

A. 可迭代对象一定是迭代器

B. 迭代器通过 **next**() 获取下一个值

C. 迭代器可以无限次重复遍历

D. 字符串不是可迭代对象



24. 以下用于创建线程的模块是（C）

A. os

B. math

C. threading

D. datetime



25. 进程和线程的区别，正确的是（C）

A. 线程是资源分配最小单位

B. 进程是 CPU 调度最小单位

C. 同一进程内线程共享资源

D. 线程比进程更占内存



26. 执行 `a = 5; b = a; a = 6`，则 b 的值是（A）

A. 5

B. 6

C. None

D. 报错

27. 下列代码返回结果是（B）

```
def outer():
    x = 10
    def inner():
        nonlocal x
        x += 5
        return x
    return inner
f = outer()
print(f())
```

A. 10

B. 15

C. 5

D. 报错

## **二、** 简答题（4题，每题 5分，共 20分）

1. 简述 Python 中for循环和while循环的区别及适用场景。

   for 循环适合遍历可迭代对象与range函数生成的等差数列

   while循环适合初始和结束条件明确的场景

2. 说明 Python 中函数参数传递的方式及特点。

   

3. 简述 Python 中模块和包的概念及作用。

4. 说明 Python 中异常处理的作用及基本语法。

5. 简述 Python 中列表、元组、字典的区别。

    列表有序可变可重复  元组有序不可变可重复 字典无序key为hash映射存储，key不可变且不可重复，value值可变可重复

6. 说明 Python 中`==`和`is`的区别。

   重写过__eq__()函数的对象== 通常判断值是否相等 未重写__eq__()函数的对象 == 作用和is相同

   is判断地址值是否相等从而确定是否为同一个对象

7. 简述 Python 中注释的作用及三种注释方式。

   

8. 说明 Python 中`input()`函数的作用及返回值类型。

   从控制台输入内容 返回一个字符串

9. 简述迭代器与生成器的区别、优缺点及使用场景。

10. 什么是装饰器？写出装饰器的基本结构与作用。

    基本结构：函数嵌套 内层函数使用了外层函数的变量 外层函数返回内层函数的函数对象 

    作用：在不改变原函数代码的基础上增强原函数功能

11. 简述 Python 中 LEGB 变量作用域规则。

12. 简述进程与线程的区别，以及 Python 多线程的 GIL 限制。

## **三、** 编程题（5题，每题 10分，共 50分）

1. 编写一个函数，接受一个整数列表作为参数，返回列表中所有偶数的和。

   def sum_ou_shu(l:list):

   ​	sum=0

   ​	for num in l:

   ​		if num%2==0

   ​		sum+=num

   ​	return sum

2. 编写程序，接收用户输入的年龄，判断并输出：未满 18 岁输出 “未成年”，18-60 岁输出 “成年”，60 岁以上输出 “老年”。

   age=int(input("请输入年龄"))

   if 0<age<18:

   ​	print(f"未成年")

   elif 18<age<60:

   ​	print(f"成年")

   else:

   ​	print(f"老年")

3. 编写一个程序，读取用户输入的一个整数，判断它是否为质数。如果是质数，输出 “是质数”；否则输出 “不是质数”。

   num=int(input("请输入一个整数"))

   if num<=1:

   ​	print(f"{num}不是质数")

   else:

   ​	for n in range(2,num**0.5+1):

   ​		if num%n==0:

   ​			print(f"{num}是质数")

   ​			break

4. 编写一个函数，接受两个字符串作为参数，返回这两个字符串拼接后的结果，并且将拼接后的字符串中的所有字母转换为大写。

   a,b=input("输入两个字符串以空格分割").split()

   c=a+b

   c.uper()

5. 编写一个程序，生成一个包含 1 到 100 之间所有能被 3 整除但不能被 5 整除的整数的列表，并输出该列表。

   l=[]

   for num in range(1,100):

   ​	if mun%3==0 and mun%5!=0:

   ​		l.append(num)

   print(l)

6. 编写一个函数，接受一个字典作为参数，字典的键是商品名称，值是商品价格。函数返回价格最高的商品名称。

   def max_price(dic:dict):

   ​	return max(dic,dic.get)

   ​	

7. 编写函数，接收一个字符串，返回该字符串的反转结果（如输入`abc`，返回`cba`）。

   def str_reverse(s):

   ​	a=s

   ​	index=a//2

   ​	for i in range(0,index):

   ​		a[0+index],a[-1-index]=a[-1-index],a[0+index]

   ​	return a

8. 编写函数，接收一个列表，返回**所有奇数的平方**组成的新列表（可用推导式）。

   def sum_squear_of_od(l):

   ​	return [i**2 for i in l if i>0 and i%2 !=0]

9. 编写一个**装饰器**，用于统计任意函数的执行时间。

   def outer(func):

   ​	count=0

   ​	def innor(func):

   ​			nonlocal count+=1

   ​			print(f"执行次数:{count}")

   ​		func()

   return func

10. 编写一个**生成器函数**，实现生成 1~n 之间所有质数。

11. 编写程序，使用多线程同时打印 1~50 和 51~100 两组数字。

12. 编写函数，接收一个嵌套列表（如 [[1,2],[3,[4,5]],6]），将其展平为一维列表。