# 函数参数分解

```python
def myfunc(x, y, z):
    print(x, y, z)

tuple_vec = (1, 0, 1)
dict_vec = {'x': 1, 'y': 0, 'z': 1}

>>> myfunc(*tuple_vec)    # 参数列表 使用 *
1, 0, 1

>>> myfunc(**dict_vec)    # 参数字典 使用 **
1, 0, 1
```

