# 一次检查多个标志

```python
x, y, z = 0, 1, 0

if x == 1 or y == 1 or z == 1:
    print('passed')

if 1 in (x, y, z):
    print('passed')

if x or y or z:
    print('passed')

if any((x, y, z)):    # x,y,z 有一个为True
    print('passed')

if all((x, y, z)):
    print('passed')    # x,y,z 全为True

# 注意 any(),all() 函数的参数为元组
```

