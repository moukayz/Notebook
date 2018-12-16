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

{% tabs %}
{% tab title="Python" %}
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
	
# 优化方法，
# 由于在计算可放入物品数量为k时的最大价值 totalValue[k][j] 时，
# 只需知道可放入物品数为 k-1 时的最大价值 totalValue[k-1][j] 和 totalValue[k-1][j-w[k]]，
# 因此我们只需要保留最大价值与背包容量的关系，即只需构建数组 totalValue[MaxWeight+1]，
# 用totalValue[w]的新值与旧值来表示放入 i 个物品的最大价值和 放入 i-1 个物品时的最大价值
# 然后用同样的方法进行迭代
def OneZero_Knapsack_DP2(Weight:list, Value:list, MaxWeight):
	size = len(Weight)
	totalValue = [0 for i in range(MaxWeight + 1)] # 初始化数组 totalValue[MaxWeight+1]

	for i in range(size):
		for j in range(MaxWeight , Weight[i]-1, -1):	# MaxWeight <= j <= Weight[i]
			totalValue[j] = max(totalValue[j], Value[i] + totalValue[j - Weight[i]])
		# 当背包当前容量小于第i个物品的重量时，i不放入背包，
		# 因此 totalValue[j < weight[i]] 的值不用改变

	return totalValue[MaxWeight]
```
{% endtab %}

{% tab title="C++" %}
```cpp
// Naive method
int MaxValue( vector<int>value, vector<int>weight, int MaxWeight, int size ) {
     if ( size == 0 ) return 0;

     if ( weight[size - 1] > MaxWeight ) {
          return MaxValue( value, weight, MaxWeight, size - 1 );
     }

     return max(
          value[size - 1] + MaxValue( value, weight, MaxWeight - weight[size - 1], size - 1 ),
          MaxValue( value, weight, MaxWeight, size - 1 )
     );

}

int OneZero_Knapsack_Naive( vector<int>value, vector<int>weight, int MaxWeight ) {
     return MaxValue( value, weight, MaxWeight, value.size() );
}

/*

Dynamic Programming approach for 0-1 knapscak

Not work when value or weight include float items !!

*/

#define index(a,b)  ((MaxWeight+1)*(a) + (b))

int OneZero_Knapsack_DP( vector<int>value, vector<int>weight, int MaxWeight ) {
     int size = value.size();
     vector<int> totalValue;
     totalValue.resize( ( MaxWeight + 1 )*( size + 1 ) );       // Use 1-D array to represent matrix tatalValue[size+1][MaxWeight+1]

     for ( int i = 0; i <= size; i++ ) {
          for ( int j = 0; j <= MaxWeight; j++ ) {
               if ( i == 0 || j == 0 )
                    totalValue[index( i, j )] = 0;
               else if ( j >= weight[i - 1] ) {
                    totalValue[index( i, j )] = max(
                         totalValue[index( i - 1, j )],         // Not include the ith item for max weight j
                         value[i - 1] + totalValue[index( i - 1, j - weight[i - 1] )] );     // Include the ith item
               }
               else
                    // Exceed the max weight, so cannot include the ith item
                    totalValue[index( i, j )] = totalValue[index( i - 1, j )];
          }
     }

     return totalValue[index( size, MaxWeight )];

}
```
{% endtab %}
{% endtabs %}

