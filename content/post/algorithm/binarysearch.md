---
title: "go-binarySearch"
date: 2019-09-26 22:40:10
tags: [binarySearch]
categories: [algorithm] 
---

[TOC]

## 14. first-position-of-target

### Description**

给定一个排序的整数数组（升序）和一个要查找的整数`target`，用`O(logn)`的时间查找到target第一次出现的下标（从0开始），如果target不存在于数组中，返回`-1`。

### **Example**

```
Example 1:
	Input:  [1,4,4,5,7,7,8,9,9,10]，1
	Output: 0
	
	Explanation: 
	the first index of  1 is 0.

Example 2:
	Input: [1, 2, 3, 3, 4, 5, 10]，3
	Output: 2
	
	Explanation: 
	the first index of 3 is 2.

Example 3:
	Input: [1, 2, 3, 3, 4, 5, 10]，6
	Output: -1
	
	Explanation: 
	Not exist 6 in array.
```

### 解决思路

典型的二分法求解,上来一看,求已经排序的数组的某个数.二分法就直接拿来用.

>  二分法定义

在[计算机科学](https://zh.wikipedia.org/wiki/计算机科学)中，**二分查找算法**（英语：binary search algorithm），也称**折半搜索算法**(英语：half-interval search algorithm)、**对数搜索算法**(英语：logarithmic search algorithm),是一种在[有序数组](https://zh.wikipedia.org/wiki/有序数对)中查找某一特定元素的搜索[算法](https://zh.wikipedia.org/wiki/算法)

```go
func BinarySearch(nums []int, target int) int {
	// write your code here
	l := 0
	r := len(nums) - 1
	for l <= r {
		mid := l + (r-l)/2
		if target == nums[mid] {
			return mid
		} else if target > nums[mid] {
			l = mid + 1
		} else {
			r = mid - 1
		}
	}
	return -1
}
```

提交之后,看到错误,定睛一看,是有重复元素导致的错误,二分法当找到某个元素的时候,立即返回.

```
输入
[3,4,5,8,8,8,8,10,13,14]
8
输出
4
期望答案
3

len(nums) = 9 ==> mid = 4
与期望的3不一样,因为num[4]=8, 下标为3.
要找的是第一个匹配的元素.算法要改进一下.
```

### 改进

```go
package lintcode

func BinarySearchX(nums []int, target int) int {
	// write your code here
	l := 0
	r := len(nums) - 1
	for l <= r {
		mid := l + (r-l)/2
		//如果大于l右移至中点后一个
		if target > nums[mid] {
			l = mid + 1
			//如果是小于等于,则r偏移至中点前一个
		} else {
			r = mid - 1
		}
	}
	//最终的l指针肯定指向target,如果不指向,说明没有找到
	if nums[l] == target {
		return l
	}
	return -1
}
```

### 测试函数

```go
package lintcode
import "testing"

func TestBinarySearchX(t *testing.T) {
	a := []int{3,4,5,8,8,8,8,10,13,14}//普通二分法失败的案例
	target := 8
	want := 3
	rel := BinarySearchX(a,target)
	if want != rel {
		t.Fatalf("want=%d, real=%d", want, rel)
	}
	t.Logf("want=%d", want)
}

=== RUN   TestBinarySearchX
--- PASS: TestBinarySearchX (0.00s)
    14-binarysearch_test.go:20: want=3
PASS
```

- 验证`[goland]`上测试模块

```
=== RUN   TestBinarySearchX
--- PASS: TestBinarySearchX (0.00s)
    14-binarysearch_test.go:20: want=3
PASS
```

- 命令行测试

```
$ go test -v 14-binarysearch_test.go  14-binarysearch.go
=== RUN   TestBinarySearchX
--- PASS: TestBinarySearchX (0.00s)
    14-binarysearch_test.go:20: want=3
PASS
ok      command-line-arguments  0.034s
```

### 复杂度分析

- [时间复杂度](https://zh.wikipedia.org/wiki/时间复杂度)

二分搜索每次把搜索区域减少一半，时间复杂度为O(log n)。（n代表集合中元素的个数）

- [空间复杂度](https://zh.wikipedia.org/wiki/空间复杂度)

O(1).虽以递归形式定义，但是[尾递归](https://zh.wikipedia.org/wiki/尾递归)，可改写为循环。

感谢您的阅读~

### 参考

- [wiki百科](https://zh.wikipedia.org/wiki/二分搜索算法)

