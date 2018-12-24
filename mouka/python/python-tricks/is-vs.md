# "is" vs "=="

{% hint style="info" %}
**当两个变量的值相同时，“==”表达式为 True**
{% endhint %}

{% hint style="info" %}
**当两个变量的指向的对象相同时，“is”表达式为 True**
{% endhint %}

```python
>>> a = [1, 2, 3]
>>> b = a

>>> a is b    # a,b 指向同一个列表
True
>>> a == b
True

>>> c = list(a)    
>>> a == c    # a 和 c 的值相同，都为 [1,2,3]
True
>>> a is c    # 但 a 和 c 是两个不同的对象
False
```

