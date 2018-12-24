# 函数作为变量

* 当作参数传递给其他函数
* 作为其他函数的返回值
* 赋值给变量或者存储在数据结构里

```python
>>> def myfunc(a, b):
...     return a + b
...
>>> funcs = [myfunc]
>>> funcs[0]
<function myfunc at 0x107012230>
>>> funcs[0](2, 3)
5
```



