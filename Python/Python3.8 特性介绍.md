<!-- TOC -->

- [简介](#简介)
- [海象表达式 :=](#海象表达式-)
- [仅位置参数 /](#仅位置参数-)
- [f-strings 说明符 =](#f-strings-说明符-)
- [启动异步 REPL](#启动异步-repl)
- [unittest 支持异步](#unittest-支持异步)

<!-- /TOC -->

## 简介

Python3.8 已经发布了, 官方文档看这里
[What’s New In Python 3.8](https://docs.python.org/3/whatsnew/3.8.html).

介绍一些 Python3.8 中的新特性.

## 海象表达式 :=

新的语法 `:=` 将给变量赋值, 这个变量是更大的表达式的一部分.

```python
if (n := len(a)) > 10:
  print(f"List is too long ({n} elements, expected <= 10)")
```

用在 if 判断中, 避免调用 len() 两次.

```python
discount = 0.0
if (mo := re.search(r'(\d+)% discount', advertisement)):
  discount = float(mo.group(1)) / 100.0
```

正则表达式匹配和获取结果的时候.

```python
# Loop over fixed length blocks
while (block := f.read(256)) != '':
  process(block)
```

用在 while 循环中, 可以同时取值, 并判断是否为空.

```python
[clean_name.title() for name in names
 if (clean_name := normalize('NFC', name)) in allowed_names]
```

用在列表推导中.

完整介绍看 [PEP 572](https://www.python.org/dev/peps/pep-0572).

## 仅位置参数 /

新的函数参数语法 `/` 指明有些函数参数必须被指定为位置参数, 不能被用作关键字参数.

```python
def f(a, b, /, c, d, *, e, f):
  print(a, b, c, d, e, f)
```

在上面的例子中, a 和 b 是仅位置参数, c 和 d 既可以是位置参数又可以是关键字参数,
e 和 f 必须是关键字参数.

```python
>>> def f(a, b, /, **kwargs):
...     print(a, b, kwargs)
...
>>> f(10, 20, a=1, b=2, c=3)         # a and b are used in two ways
10 20 {'a': 1, 'b': 2, 'c': 3}
```

仅位置参数的参数名在 `**kwargs` 中仍旧可用.

```python
class Counter(dict):
  def __init__(self, iterable=None, /, **kwds):
    # Note "iterable" is a possible keyword argument
```

完整介绍看 [PEP 570](https://www.python.org/dev/peps/pep-0570).

## f-strings 说明符 =

`f-strings` 增加了 `=` 说明符, `f'{expr=}'` 会被扩展为表达式的文本,
加上一个等号, 和一个执行表达式的结果.

```python
>>> user = 'eric_idle'
>>> member_since = date(1975, 7, 31)
>>> f'{user=} {member_since=}'
"user='eric_idle' member_since=datetime.date(1975, 7, 31)"
```

## 启动异步 REPL

使用 `python -m asyncio` 启动一个异步的 REPL,
可以直接在 top-level 级别使用 `await`,
不用在封装到函数中了.

## unittest 支持异步

```python
import unittest

class TestRequest(unittest.IsolatedAsyncioTestCase):

    async def asyncSetUp(self):
        self.connection = await AsyncConnection()

    async def test_get(self):
        response = await self.connection.get("https://example.com")
        self.assertEqual(response.status_code, 200)

    async def asyncTearDown(self):
        await self.connection.close()


if __name__ == "__main__":
    unittest.main()
```
