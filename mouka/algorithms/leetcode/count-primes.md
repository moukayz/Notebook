# Count Primes

 Count the number of prime numbers less than a non-negative number, **n**.

### Example:

```text
Input: 10
Output: 4
Explanation: There are 4 prime numbers less than 10, they are 2, 3, 5, 7.
```

### Solution:

```python
def countPrimes(self, n):
        """
        :type n: int
        :rtype: int
        """
        arr = [True] * n
        upper = n**0.5 + 1
        i = 2
        while i < upper:
            for j in range(i*i,n,i):
                arr[j] = False
            i = i + 1
                
        return arr[2:].count(True)
```

