### 一、选择题

- 以下哪种异常通常在尝试访问字典中不存在的键时引发？（   ）

  A. KeyError	B. IndexError C. ValueError	D. TypeError

- 以下关于try - except - finally语句的描述，正确的是（   ）

  A. finally块中的代码只有在没有异常发生时才会执行	B. finally块中的代码无论是否发生异常都会执行

  C. 如果try块中发生异常，except块和finally块都不会执行	D. except块和finally块只能存在一个

### 二、编程题

1. 编写一段 Python 代码，尝试将字符串 "123abc" 转换为整数，如果转换失败，捕获 ValueError 异常，将异常信息记录到一个文本文件 error.log 中。

2. 定义一个函数check_age，该函数接受一个年龄参数。如果年龄小于 0，抛出一个自定义异常InvalidAgeError；如果年龄大于 120，抛出UnrealisticAgeError。这两个自定义异常类都继承自Exception类。调用该函数并传入一个不合法的年龄值，捕获并处理异常。

3. 编写函数`get_mondays(start_date: str, n: int)`接收一个起始日期（字符串格式 `YYYY-MM-DD`）和天数 N，完成：

- 把字符串转为 datetime 日期对象
- 生成从起始日期开始 **未来 N 天** 的所有日期
- 筛选出其中所有**周一**的日期
- 返回周一日期列表

```
输入：start_date="2025-01-01", N=30
输出：["2025-01-06", "2025-01-13", "2025-01-20", "2025-01-27"]
```

