---
layout:		post
title:		"生成排列的算法汇总"
subtitle: 	"彻底掌握如何生成排列"
date:		2019-10-15T12:30:00
author:		"darjun"
image:	"img/post-bg-2015.jpg"
tags:
    - 算法
URL: "2019/10/15/algorithm/permutation"
categories: [
	"Algorithms"
]
---

## 概述

我觉得自己的算法思维能力有些薄弱，所以基本上每天晚上都会抽空做1-2到 leetcode 算法题。这两天遇到一个排列的问题——[Next Permutation](https://leetcode.com/problems/next-permutation/)。然后我去搜索了一下生成排列的算法。这里做一下总结。

## 算法

目前，生成一个序列的排列常用的有以下几种算法：

* 暴力法（Brute Force）
* 插入法（Insert）
* 字典法（Lexicographic）
* SJT算法（Steinhaus-Johnson-Trotter）
* 堆算法（Heap）

下面依次介绍算法的内容，实现和优缺点。

在介绍这些算法之前，我们先做一些示例和代码上的约定：

* 我的代码实现是使用 Go 语言，且仅实现了求`int`切片的所有排列，其它类型自行扩展也不难。
* 除非特殊说明，我假定输入的`int`中无重复元素，有重复元素可自行去重，其中有个别算法可处理重复元素的问题。

完整代码放在[Github](https://github.com/darjun/algorithms/tree/master/golang/permutation)上。

### 暴力法

#### 描述

**暴力法**是很直接的一种分治法：先生成 n-1 个元素的排列，加上第 n 个元素即可得到 n 个元素的排列。算法步骤如下：

* 将第 n 个元素依次交换到最后一个位置上
* 递归生成前 n-1 个元素的排列
* 加上最后一个元素即为 n 个元素的排列

#### 实现

算法实现也很简单。这里引入两个辅助函数，拷贝和反转切片，后面代码都会用到：

```golang
func copySlice(nums []int) []int {
  n := make([]int, len(nums), len(nums))
  copy(n, nums)
  return n
}

// 反转切片nums的[i, j]范围
func reverseSlice(nums []int, i, j int) {
  for i < j {
    nums[i], nums[j] = nums[j], nums[i]
    i++
    j--
  }
}
```

算法代码如下：

```golang
func BruteForce(nums []int, n int, ans *[][]int) {
  if n == 1 {
    *ans = append(*ans, copySlice(nums))
    return
  }

  n := len(nums)
  for i := 0; i < n; i++ {
    nums[i], nums[n-1] = nums[n-1], nums[i]
    BruteForce(nums, n-1, ans)
    nums[i], nums[n-1] = nums[n-1], nums[i]
  }
}
```

作为一个接口，需要做到尽可能简洁，第二个参数初始值就是前一个参数切片的长度。优化接口：

```golang
func bruteForceHelper(nums []int, n int, ans *[][]int) {
  // 生成排列逻辑
  ...
}

func BruteForce(nums []int) [][]int{
  ans := make([][]int, 0, len(nums))
  bruteForceHelper(nums, len(nums), &ans)
  return ans
}
```

#### 优缺点

优点：逻辑简单直接，易于理解。

缺点：返回的排列数肯定是`n!`，性能的关键在于系数的大小。由于暴力法的每次循环都需要交换两个位置上的元素，递归结束后又需要**再交换回来**，在`n`较大的情况下，性能较差。

### 插入法

#### 描述

插入法顾名思义就是将元素插入到一个序列中所有可能的位置生成新的序列。从 1 个元素开始。例如要生成`{1,2,3}`的排列：

* 先从序列 1 开始，插入元素 2，有两个位置可以插入，生成两个序列 12 和 21
* 将 3 插入这两个序列的所有可能位置，生成最终的 6 个序列

```
          1
    12          21
123 132 312  213 231 321
```

#### 实现

实现如下：

```golang
func insertHelper(nums []int, n int) [][]int {
  if n == 1 {
    return [][]int{[]int{nums[0]}}
  }

  var ans [][]int
  for _, subPermutation := range insertHelper(nums, n-1) {
    // 依次在位置0-n上插入
    for i := 0; i <= len(subPermutation); i++ {
      permutation := make([]int, n, n)
      copy(permutation[:i], subPermutation[:i])
      permutation[i] = nums[n-1]
      copy(permutation[i+1:], subPermutation[i:])
      ans = append(ans, permutation)
    }
  }

  return ans
}

func Insert(nums []int) [][]int {
  return insertHelper(nums, len(nums))
}
```

#### 优缺点

优点：同样是简单直接，易于理解。

缺点：由于算法中有不少的数据移动，性能与暴力法相比降低了**16%**。

### 字典法

#### 描述

**该算法有个前提是序列必须是有升序排列的**，当然也可以微调对其它序列使用。它通过修改当前序列得到下一个序列。我们为每个序列定义一个**权重**，类比序列组成的数字的大小，序列升序排列时**“权重”**最小，降序排列时**“权重”**最大。下面是 1234 的排列按**“权重”由小到大：

```
1234
1243
1324
1342
1423
1432
2134
...
```

我们观察到一开始最高位都是 1，稍微调整一下后面三个元素的顺序就可以使得整个**“权重”**增加，类比整数。当后面三个元素已经逆序时，下一个序列最高位就必须是 2 了，因为仅调整后三个元素已经无法使**“权重”**增加了。算法的核心步骤为：

* 对于当前的序列，找到索引`i`满足其后的元素完全逆序。
* 这时索引`i`处的元素需要变为后面元素中大于该元素的最小值。
* 然后剩余元素升序排列，即为当前序列的下一个序列。

该算法用于 C++ 标准库中`next_permutation`算法的实现，见[GNU C++ std::next_permutation](https://gcc.gnu.org/onlinedocs/libstdc++/libstdc++-html-USERS-4.4/a01347.html)。


#### 实现

```golang
func NextPermutation(nums []int) bool {
	if len(nums) <= 1 {
		return false
	}

	i := len(nums) - 1
	for i > 0 && nums[i-1] > nums[i] {
		i--
	}

	// 全都逆序了，达到最大值
	if i == 0 {
		reverse(nums, 0, len(nums)-1)
		return false
	}

	// 找到比索引i处元素大的元素
	j := len(nums) - 1
	for nums[j] <= nums[i-1] {
		j--
	}

	nums[i-1], nums[j] = nums[j], nums[i-1]
	// 将后面的元素反转
	reverse(nums, i, len(nums)-1)
	return true
}

func lexicographicHelper(nums []int) [][]int {
	ans := make([][]int, 0, len(nums))
	ans = append(ans, copySlice(nums))
	for NextPermutation(nums) {
		ans = append(ans, copySlice(nums))
	}

	return ans
}

func Lexicographic(nums []int) [][]int {
	return lexicographicHelper(nums)
}
```

`NextPermutation`函数即可用于解决前文 LeetCode 算法题。其返回`false`表示已经到达最后一个序列了。

#### 优缺点

优点：`NextPermutation`可以单独使用，性能也不错。

缺点：稍微有点难理解。

### SJT算法

#### 描述

SJT 算法在前一个排列的基础上通过**仅交换相邻的两个元素**来生成下一个排列。例如，按照下面顺序生成 123 的排列:

```
123(交换23) ->
132(交换13) ->
312(交换12) ->
321(交换32) ->
231(交换31) ->
213
```

一个简单的方案是通过 n-1 个元素的排列生成 n 个元素的排列。例如我们现在用 2 个元素的排列生成 3 个元素的排列。

2 个元素的排列只有 2 个： 1 2 和 2 1。

通过在 2 个元素的排列中所有不同的位置插入 3，我们就能得到 3 个元素的排列。

在 1 2 的不同位置插入 3 得到：1 2 **3**，1 **3** 2 和 **3** 1 2。 在 2 1 的不同位置插入 3 得到：2 1 **3**，2 **3** 1 和 **3** 2 1。

上面是插入法的逻辑，但是插入法由于有大量的数据移动导致性能较差。SJT 算法不要求生成所有 n-1 个元素的排列。它记录排列中每个元素的方向。算法步骤如下：

* 查找序列中可移动的最大元素。一个元素可移动意味着它的值大于它指向的相邻元素。
* 交换该元素与它指向的相邻元素。
* 修改所有值大于该元素的元素的方向。
* 重复以上步骤直到没有可移动的元素。

假设我们需要生成序列 1 2 3 4 的所有排列。首先初始化所有元素的方向为从右到左。第一个排列即为初始序列：

```
<1 <2 <3 <4
```

所有可移动的元素为 2，3 和 4。最大的为 4。我们交换 3 和 4。由于此时 4 是最大元素，不用改变方向。得到下一个排列：

```
<1 <2 <4 <3
```

4 还是最大的可移动元素，交换 2 和 4，不用改变方向。得到下一个排列：

```
<1 <4 <2 <3
```

4 还是最大的可移动元素，交换 1 和 4，不用改变方向。得到下一个排列：

```
<4 <1 <2 <3
```

当前 4 已经无法移动了，3 成为最大的可移动元素，交换 2 和 3。注意，元素 4 比 3 大，所以要改变元素 4 的方向。得到下一个排列：

```
>4 <1 <3 <2
```

这时，元素 4 又成为了最大的可移动元素，交换 4 和 1。注意，此时元素 4 方向已经变了。得到下一个排列：

```
<1 >4 <3 <2
```

交换 4 和 3，得到下一个排列：

```
<1 <3 >4 <2
```

交换 4 和 2：

```
<1 <3 <2 >4
```

这时元素 3 为可移动的最大元素，交换 1 和 3，改变元素 4 的方向：

```
<3 <1 <2 <4
```

继续这个过程，最后得到的排列为（**强烈建议自己试试**）：

```
<2 <1 >3 >4
```

已经没有可移动的元素了，算法结束。

#### 实现

```golang
func getLargestMovableIndex(nums []int, dir []bool) int {
	maxI := -1
	l := len(nums)
	for i, num := range nums {
		if dir[i] {
			if i > 0 && num > nums[i-1] {
				if maxI == -1 || num > nums[maxI] {
					maxI = i
				}
			}
		} else {
			if i < l-1 && num > nums[i+1] {
				if maxI == -1 || num > nums[maxI] {
					maxI = i
				}
			}
		}
	}

	return maxI
}

func sjtHelper(nums []int, ans *[][]int) {
	l := len(nums)
	// true 表示方向为从右向左
	// false 表示方向为从左向右
	dir := make([]bool, l, l)
	for i := range dir {
		dir[i] = true
	}

	maxI := getLargestMovableIndex(nums, dir)
	for maxI >= 0 {
		maxNum := nums[maxI]
		// 交换最大可移动元素与它指向的元素
		if dir[maxI] {
			nums[maxI], nums[maxI-1] = nums[maxI-1], nums[maxI]
			dir[maxI], dir[maxI-1] = dir[maxI-1], dir[maxI]
		} else {
			nums[maxI], nums[maxI+1] = nums[maxI+1], nums[maxI]
			dir[maxI], dir[maxI+1] = dir[maxI+1], dir[maxI]
		}

		*ans = append(*ans, copySlice(nums))

		// 改变所有大于当前移动元素的元素的方向
		for i, num := range nums {
			if num > maxNum {
				dir[i] = !dir[i]
			}
		}

		maxI = getLargestMovableIndex(nums, dir)
	}
}

func Sjt(nums []int) [][]int {
	ans := make([][]int, 0, len(nums))
	ans = append(ans, copySlice(nums))
	sjtHelper(nums, &ans)
	return ans
}
```

#### 优缺点

优点：作为一种算法思维可以学习借鉴。

缺点：性能不理想。

### Heap算法

#### 描述

Heap算法优雅、高效。它是从暴力法演化而来的，我们前面提到暴力法性能差主要是由于多次交换，堆算法就是通过减少交换提升效率。

算法步骤如下：

* 如果元素个数为奇数，交换第一个和最后一个元素。
* 如果元素个数为偶数，依次交换第 i 个和最后一个元素。

[Wikipedia](https://en.wikipedia.org/wiki/Heap%27s_algorithm)上有详细的证明，有兴趣可以看看。

#### 实现

```golang
func heapHelper(nums []int, n int, ans *[][]int) {
	if n == 1 {
		*ans = append(*ans, copySlice(nums))
		return
	}

	for i := 0; i < n-1; i++ {
		heapHelper(nums, n-1, ans)
		if n&1 == 0 {
			// 如果是偶数，交换第i个与最后一个元素
			nums[i], nums[n-1] = nums[n-1], nums[i]
		} else {
			// 如果是奇数，交换第一个与最后一个元素
			nums[0], nums[n-1] = nums[n-1], nums[0]
		}
	}
	heapHelper(nums, n-1, ans)
}

// Heap 使用堆算法生成排列
func Heap(nums []int) [][]int {
	ans := make([][]int, 0, len(nums))
	heapHelper(nums, len(nums), &ans)
	return ans
}
```

Heap 算法非常难理解，而且很容易写错，我现在纯粹是背下来了😂 。

#### 优缺点

优点：代码实现简单、高效。

缺点：非常难理解。

## 结语

本文介绍了几种生成排列的算法，希望对大家有所帮助。