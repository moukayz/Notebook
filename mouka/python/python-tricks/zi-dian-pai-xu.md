# 字典排序

```python
>>> xs = {'a': 4, 'b': 3, 'c': 2, 'd': 1}

>>> sorted(xs.items(), key=lambda x: x[1])    # 按value排序
[('d', 1), ('c', 2), ('b', 3), ('a', 4)]
>>> sorted(xs.items(), key=lambda x: x[0])    # 按key排序    
[('a', 4), ('b', 3), ('c', 2), ('d', 1)]    
# Or:

>>> import operator
>>> sorted(xs.items(), key=operator.itemgetter(1))
[('d', 1), ('c', 2), ('b', 3), ('a', 4)]
>>> sorted(xs.items(),key=operator.itemgetter(0))
[('a', 4), ('b', 3), ('c', 2), ('d', 1)]
```



