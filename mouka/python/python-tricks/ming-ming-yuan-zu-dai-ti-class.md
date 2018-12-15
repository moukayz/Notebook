# 命名元组代替class

{% hint style="info" %}
使用命名元组 **namedtuple** 可以代替简单的class定义
{% endhint %}

```python
>>> from collections import namedtuple
>>> Car = namedtuple('Car', 'color mileage')

>>> my_car = Car('red', 3812.4)
>>> my_car.color
'red'
>>> my_car.mileage
3812.4

# 打印命名元组信息
>>> my_car
Car(color='red' , mileage=3812.4)

# 命名元组同元组（tuple）一样，无法修改（immutable）
>>> my_car.color = 'blue'
AttributeError: "can't set attribute"
```

