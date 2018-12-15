# 计算代码执行时间

{% hint style="info" %}
`timeit 模块可用来测量`**`少量代码`**`的执行时间` **``**
{% endhint %}

```python
>>> import timeit
>>> timeit.timeit('"-".join(str(n) for n in range(100))',
                  number=10000)
0.3412662749997253

>>> timeit.timeit('"-".join([str(n) for n in range(100)])',
                  number=10000)
0.2996307989997149

>>> timeit.timeit('"-".join(map(str, range(100)))',
                  number=10000)
0.24581470699922647
```

