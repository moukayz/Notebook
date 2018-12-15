# 使括号平衡的最小交换次数

### 描述：

给定 2N 长度的字符串，包含N个 '\[' ，N个 '\]'，计算使字符串平衡的最小交换次数

平衡字符串定义：S1\[S2\]， 其中 S1，S2均为平衡字符串

### 输入： 

2N长度字符串

### 输出：

最小交换次数

### 举例：

> Input : \[\]\]\[\]\[ 
>
> Output : 2 
>
> First swap: Position 3 and 4 \[\]\[\]\]\[ 
>
> Second swap: Position 5 and 6 \[\]\[\]\[\]
>
>
>
> Input : \[\[\]\[\]\] 
>
> Output : 0 
>
> String is already balanced.

{% hint style="info" %}
Solution is below
{% endhint %}

### 思路

顺序遍历字符串，当遇到不匹配的'\]'时（遇到相应的'\['之前），将其与之后最接近它的'\]'交换位置

### 实现

```python
def MinSwapBracketBalancing(Brackets: list):
	# 首先遍历字符串，获取所有左括号的位置
	leftBrackets = []
	leftBrackets.extend([ idx for idx, x in enumerate(Brackets) if x == '['])

	if len(leftBrackets) != len(Brackets) // 2:	
		return 0

	count = 0	# 当前还未平衡的左括号数量
	pos = 0		# 下一个左括号的位置
	sum = 0		# 总共移动次数

	i = 0
	while i < len(Brackets):
		if Brackets[i] == '[':
			# 当前遍历括号为左括号
			count += 1
			pos += 1
		else:
			# 当前括号为右括号
			count -= 1	# 不平衡左括号 - 1
			if count < 0:
				# 右括号在不平衡左括号之前出现
				# 交换两括号位置
				Brackets[i], Brackets[leftBrackets[pos]] = Brackets[leftBrackets[pos]],Brackets[i]

				sum += leftBrackets[pos] - i
				count = 0
				pos += 1
				i += 1
		i += 1

	return sum
```

