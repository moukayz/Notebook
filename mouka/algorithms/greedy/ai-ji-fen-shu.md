# 埃及分数

### 描述：

每一个分数都可以表达为多个不同的单元分数的和， 

单元分数： 分子为1且分母为正整数

### 举例： 

2/3 --&gt; 1/2 + 1/6 

6/14 --&gt; 1/3 + 1/11 + 1/231 

12/13 --&gt; 1/2 + 1/3 + 1/12 + 1/156

### 输入： 

a,b （a为输入分数的分子， b为输入分数的分母， **假定分母大于等于分子**）

### 输出： 

对应单元分数的分母序列

> Input : 2,3 
>
> Output : \[2,6\]

{% hint style="info" %}
Solution is below !
{% endhint %}

### 思路

（分子 a， 分母 b）：

1. 如果 b 能整除 a，直接返回 b/a
2. 如果 b 不能整除 a，则采用贪心算法的思想，每次求出最接近 a/b 的单元分数（假定为c），再使用递归的方法求得余下部分 a/b - c 的单元分数

### 实现

```python
def EgyptianFraction(numerator, denominator):
	result = []

	if denominator % numerator == 0:
		result.append(denominator // numerator)
	elif numerator < denominator:
		# 计算出最大单元分数
		unit = denominator // numerator + 1
		result.append(unit)

		# 求出余下部分的单元分数 a/b - 1/c = (a*c-1) / (b*c)
		remain = EgyptianFraction(numerator * unit - denominator,denominator * unit)
		result.extend(remain)
	# 不考虑其他情况

	return result
```



