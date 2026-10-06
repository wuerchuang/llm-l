## 题1：迭代器

定义一个迭代器类 `RangeIterator`，接收两个参数 `start`、`end`，实现功能：

可以遍历 **[start, end) 之间所有 3 的倍数**，必须手动实现 `__iter__` 和 `__next__` 方法，禁止用生成器。

## 题2：生成器

编写一个**生成器函数** `fib_generator(n)`，生成**前 n 项斐波那契数列**。规则：

- 第 1、2 项为 1
- 从第 3 项开始，每一项 = 前两项之和
- 使用 `yield` 实现，不能一次性返回列表

## 题3：装饰器

编写一个**装饰器** `timer`，可以装饰任意函数，**自动打印该函数的运行时间（秒）**。要求：

- 装饰器不带参数
- 能装饰有参 / 无参函数
- 输出格式：`函数名 运行时间：x.xx 秒`，提示：`函数对象.__name__`可以获取函数名

例如：被装饰函数如下：

```python
def test_func():
    print("hello")
   
def add(start,end):
    sum = 0
    for i in range(start,end+1):
        sum += i
    return sum
```



