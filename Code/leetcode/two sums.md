дана последовательность и число-цель. нужно достать из последовательности такие два числа, которые в сумме дадут число-цель, и вернуть их индексы в последовательности.
например:
```
ввод: my_method([3, 2, 4], 6)
вывод: [1, 2]
```
самый простой способ реализовать такую историю:
```Python
def my_method(nums: List[int], target: int):
	for i in range(len(nums)):
		for j in range(i + 1, len(nums)):
			result = nums[i] + nums[j]
			if result == target:
				return [i, j]
```
однако временная сложность такого алгоритма квадратичная - $O(n^2)$, поэтому он не очень хороший.
можно уложиться в константное время, если попробовать решить задачу через хэш-таблицу:
```Python
def my_method(nums: List[int], target: int):
	hashmap = {}
	for i in range(len(nums)):
		hashmap(nums[i]) = i
	for i in range(len(nums)):
		result = target - nums[i]
		if result in hashmap and hashmap[result] != i:
			return [i, hashmap[result]]
```