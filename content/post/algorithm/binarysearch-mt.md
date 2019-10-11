---
title: "Mounting Array"
date: 2019-09-26 22:40:10
tags: [binarySearch]
categories: [algorithm] 
---

[TOC]

## 1095.[ 山脉数组中查找目标值](https://leetcode-cn.com/problems/find-in-mountain-array/)

### 描述

给你一个 **山脉数组** `mountainArr`，请你返回能够使得 `mountainArr.get(index)` **等于** `target` **最小** 的下标 `index` 值。

如果不存在这样的下标 `index`，就请返回 `-1`。

所谓山脉数组，即数组 `A` 假如是一个山脉数组的话，需要满足如下条件：

**首先**，`A.length >= 3`

**其次**，在 `0 < i < A.length - 1` 条件下，存在 `i` 使得：

- `A[0] < A[1] < ... A[i-1] < A[i]`
- `A[i] > A[i+1] > ... > A[A.length - 1]`

你将 **不能直接访问该山脉数组**，必须通过 `MountainArray` 接口来获取数据：

- `MountainArray.get(k)` - 会返回数组中索引为`k` 的元素（下标从 0 开始）
- `MountainArray.length()` - 会返回该数组的长度

示例 1：

```
输入：array = [1,2,3,4,5,3,1], target = 3
输出：2
解释：3 在数组中出现了两次，下标分别为 2 和 5，我们返回最小的下标 2。
```
示例 2：

```
输入：array = [0,1,2,4,2,1], target = 3
输出：-1
解释：3 在数组中没有出现，返回 -1。
```
**提示**

1. `3 <= mountain_arr.length() <= 10000`
2. `0 <= target <= 10^9`
3. `0 <= mountain_arr.get(index) <= 10^9`

### 解法思路

根据山脉数组的特点,想找的值只能在上升或者下降的某个点,可以先找到山脉数组的最大值,利用二分查找的特点,找到最大值的下标,将数组分为两个部分,一个递增数组和递减数组.再在递增数组里面找想要找的值,找不到在递减数组里面找,否则就返回-1.

### 实现代码

```
/*
Copyright 2019 louis.
@Time : 2019/9/30 20:36
@Author : louis
@File : 1095-findinMountingarray
@Software: GoLand

*/

package leetcode

type MountainArray struct {
	arr []int
}

func (m *MountainArray) get(index int) int {
	return m.arr[index]
}

func (m *MountainArray) length() int {
	return len(m.arr)
}

func bs(m *MountainArray, t, l, r int, asc bool) int {
	for l < r {
		mid := l + (r-l)>>1
		if (asc && m.get(mid) >= t) || (!asc && m.get(mid) <= t) {
			r = mid
		} else {
			l = mid + 1
		}
	}
	if t == m.get(l) {
		return l
	}
	return -1
}

func FindInMountainArray(target int, m *MountainArray) int {
	p, r := 0, m.length()-1
	for p < r {
		mid := p + (r-p)>>1
		// 如果mid值大于mid+1;说明峰值在mid的左边.否则只能在右边
		if m.get(mid) > m.get(mid+1) {
			r = mid
		} else {
			p = mid + 1
		}
	}
	//此次循环找到了p对应是峰值.左边进行查找,优先查找下标小的.
	i := bs(m, target, 0, p-1, true)
	if i != -1 {
		return i
	}
	//在右边进行查找
	return bs(m, target, p, m.length()-1, false)
}

```

### test函数

```
/*
Copyright 2019 louis.
@Time : 2019/9/30 20:36
@Author : louis
@File : 1095-findinMountingarray_test.go
@Software: GoLand

*/

package leetcode

import "testing"

func TestFindInMountainArray(t *testing.T) {
	type args struct {
		target int
		m      *MountainArray
	}
	tests := []struct {
		name string
		args args
		want int
	}{
		{"test01",args{3,&MountainArray{[]int{1,2,3,4,5,3,1}}},2},
		{"test02",args{3,&MountainArray{[]int{1,2,5,7,9,2,1}}},-1},
	}
	for _, tt := range tests {
		t.Run(tt.name, func(t *testing.T) {
			if got := FindInMountainArray(tt.args.target, tt.args.m); got != tt.want {
				t.Errorf("FindInMountainArray() = %v, want %v", got, tt.want)
			}
		})
	}
}

```

测试

```
go test -v 1095-findinMountingarray.go 1095-findinMountingarray_test.go
=== RUN   TestValidMountingArray
--- PASS: TestValidMountingArray (0.00s)
=== RUN   TestValidMountingArray/test01
    --- PASS: TestValidMountingArray/test01 (0.00s)
=== RUN   TestValidMountingArray/test02
    --- PASS: TestValidMountingArray/test02 (0.00s)
PASS
```

