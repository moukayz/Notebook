# 0-1 背包问题

### 描述：

给定 n 个物品的 重量 weight，价值 value，以及一个容量为 W 的背包，求如何装入物品，使背包内物品价值最大

**对于一件物品，只能选择或不选择（0-1）**

### 输入：

* value\[n\]   —— n 个物品的价值表 
*  weight\[n\] —— n 个物品的重量表 
* MaxWeight  —— 背包的最大容量

### 输出： 

* MaxValue —— 背包可获得的最大价值 
* items —— 选择装入背包的物品列表

### 举例： 

> Input: 
>
> val\[\] = {60, 100, 120} 
>
> wt\[\] = {10, 20, 30} 
>
> MaxWeight = 50
>
> Output: MaxValue = 220
>
>  items = \[1,2\]

{% hint style="info" %}
Solution is below!
{% endhint %}

### 思路

使用二维数组 map\[N\]\[W\] ，N 为放入背包的最大物品个数， W 为背包中物品的最大重量， 从 map\[0\]\[0\]开始构建数组，（map\[0\]\[j\] = 0, map\[i\]\[0\] = 0）

对于 map\[i\]\[j\]，其可能取值只有两种情况： 

1. 第i个物品不放人背包，则map\[i\]\[j\] = map\[i-1\]\[j\] 
2. 第i个物品放入背包， 则 map\[i\]\[j\] = map\[i-1\]\[j-weight\[i\]\] + value\[i\]

当 weight\[i\] 小于当前最大背包容量 j 时，只能取第1种

### 实现

```python
def OneZero_Knapsack_DP(Weight:list, Value:list, MaxWeight):
	size = len(Weight)
	# 初始化二维数组，加1是增加为0的情况
	totalValue = [[0] * (MaxWeight + 1) for i in range(size + 1)]

	for i in range(size + 1):
		for j in range(MaxWeight + 1):
			if i == 0 or j == 0:
				totalValue[i][j] = 0
			elif j >= Weight[i - 1]:
				# 第i个物品可以放入背包
				# 取二者最大值
				totalValue[i][j] = max(totalValue[i - 1][j], 
					totalValue[i - 1][j - Weight[i - 1]] + Value[i - 1])
			else:
				# 第i个物品不能放入背包
				totalValue[i][j] = totalValue[i - 1][j]

	return totalValue[size][MaxWeight]
```

