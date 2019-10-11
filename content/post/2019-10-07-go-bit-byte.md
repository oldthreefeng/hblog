---
title: "Go 位运算详解"
date: 2019-09-17 00:40
tags: [bit,leetcode]
categories: [algorithm] 
---

[TOC]

> ### 背景
>
> 位运算一直都懵懵懂懂,需要记录成文字.时常看看

## 位操作


1. 判断某一位是否为1
2. 只改变其中某一位,而保持其他位不变

- 按位与`&`

```cgo
& 只有两个二进制位均为1,结果才是1;其他都是0.
例如: 获取某些变量中的某一位,某些位清0且同事保留其他位不变;
n = n & 0xffffff00  (低8位置0)

如何判断int型变量n的第7位,(右往左,从0开始).
```

- 按位或`|`

```cgo
| 只有两个二进制有一个1,结果会为1,其他都是0
例如: 通过用来将变量的某些为置1保留其他位不变
n|= 0xff (将n的低8位置1)

```
- 按位异或`^`

```cgo
^ 只有两个二进制位不相同,结果才为1,其他都是0
例如: 将某些变种的某些位进行取反,且保留其他位
特点 如果a^b=c, 那么就有 c^b=a ,c^a=b (穷举法)

```
假定 A 为60，B 为13

| 运算符 | 描述                                                         | 实例                                   |
| :----- | :----------------------------------------------------------- | :------------------------------------- |
| &      | 按位与运算符"&"是双目运算符。 其功能是参与运算的两数各对应的二进位相与。 | (A & B) 结果为 12, 二进制为 0000 1100  |
| \|     | 按位或运算符"\|"是双目运算符。 其功能是参与运算的两数各对应的二进位相或 | (A \| B) 结果为 61, 二进制为 0011 1101 |
| ^      | 按位异或运算符"^"是双目运算符。 其功能是参与运算的两数各对应的二进位相异或，当两对应的二进位相异时，结果为1。 | (A ^ B) 结果为 49, 二进制为 0011 0001  |
| <<     | 左移运算符"<<"是双目运算符。左移n位就是乘以2的n次方。 其功能把"<<"左边的运算数的各二进位全部左移若干位，由"<<"右边的数指定移动的位数，高位丢弃，低位补0。 | A << 2 结果为 240 ，二进制为 1111 0000 |
| >>     | 右移运算符">>"是双目运算符。右移n位就是除以2的n次方。 其功能是把">>"左边的运算数的各二进位全部右移若干位，">>"右边的数指定移动的位数。 | A >> 2 结果为 15 ，二进制为 0000 1111  |

## 示例

- 获取c的第i位的bit值.

```go
func GetBit(c byte, i uint) byte {
	return (c >> i) & 0x1
}
```

- 将c的第i位设置为v.

```go
func SetBit(c byte, i uint, v int) byte {
	if v != 0 {
		/**
		将某一位设置为1，例如设置第8位，从右向左数需要偏移7位,注意不要越界
		1<<7=1000 0000 然后与c逻辑或|,偏移后的第8位为1，逻辑|运算时候只要1个为真就为真达到置1目的
		*/
		c |= 1 << i
	} else {
		/**
		将某一位设置为0，例如设置第4位，从右向左数需要偏移3位,注意不要越界
		1<<3=0000 1000 然后取反得到 1111 0111 然后逻辑&c
		*/
		c = &^ (1 << i)
	}
}
```

- 将c的第i位翻转

```go
func FlipBit(c byte, i uint) byte {
	c = c ^ (1 << i)
	/*
	将第4位翻转, 1左移3位, 1 << 3 ==> 0000 1000
	0000 1000 ^ 0001 1110 == 0001 0110
	*/
	return c
}
```

### 231. 判断为2的幂

- 位运算

```go
func IsPowerOfTwo(n int) bool {
	if n <= 0 {
		return false
	}
    /*
    n位2的幂时, 	2^7   == 1000 0000
    n-1为 	   	  2^7-1 == 0111 1111
    n & (n-1)必定全为,	  == 0000 0000 
    */
    return (n & (n-1)) == 0
}
```

- 普通递归

```go
func IsPowerOfTwo(n int) bool {
	if n <= 0 {
		return false
	}
    // 2的幂一直除2, 最后等于1
	if n == 1 {
		return true
	}
    // n为奇数,必定不为2的幂
	if n % 2 != 0 { 
		return false
	} 
	return isPowerOfTwo(n / 2)
}
```

#### 验证

```go
package main

import (
	"fmt"
	"github.com/imroc/biu"
)

func main() {
	var a byte
	a = 30 //00011110
	b := GetBit(a, 2)
	fmt.Printf("%d二进制为:%v\n", a, biu.ToBinaryString(a))
	fmt.Printf("a的第2位为:%v\n", b)
	b = SetBit(a, 5, 1)
	fmt.Printf("%d的第6位,置为1后的二进制为:%v\n", a, biu.ToBinaryString(b))
	b = SetBit(a, 6, 1)
	fmt.Printf("%d的第7位,置为1后的二进制为:%v\n", a, biu.ToBinaryString(b))
	b = SetBit(a, 7, 1)
	fmt.Printf("%d的第8位,置为1后的二进制为:%v\n", a, biu.ToBinaryString(b))
	b = SetBit(a, 0, 1)
	fmt.Printf("%d的第1位,置为1后的二进制为:%v\n", a, biu.ToBinaryString(b))
	b = SetBit(a, 1, 0)
	fmt.Printf("%d的第2位,置为0后的二进制为:%v\n", a, biu.ToBinaryString(b))

	for i := 0; i < 20; i++ {
		if IsPowerOfTwo(i) {
			fmt.Printf("%d \tisPowerOfTwo\n", i)
		}
	}
}
```

- output

```
30二进制为:00011110
a的第2位为:1
30的第6位,置为1后的二进制为:00111110
30的第7位,置为1后的二进制为:01011110
30的第8位,置为1后的二进制为:10011110
30的第1位,置为1后的二进制为:00011111
30的第2位,置为0后的二进制为:00011100
1 	isPowerOfTwo
2 	isPowerOfTwo
4 	isPowerOfTwo
8 	isPowerOfTwo
16 	isPowerOfTwo
```

### 5213. Play with chips

#### 描述

数轴上放置了一些筹码，每个筹码的位置存在数组 chips 当中。

你可以对 任何筹码 执行下面两种操作之一（不限操作次数，0 次也可以）：

- 将第 i 个筹码向左或者右移动 2 个单位，代价为 0。
- 将第 i 个筹码向左或者右移动 1 个单位，代价为 1。

最开始的时候，同一位置上也可能放着两个或者更多的筹码。

返回将所有筹码移动到同一位置（任意位置）上所需要的最小代价。

示例 1：

```
输入：chips = [1,2,3]
输出：1
解释：第二个筹码移动到位置三的代价是 1，第一个筹码移动到位置三的代价是 0，总代价为 1。
```
示例 2：

```
输入：chips = [2,2,2,3,3]
输出：2
解释：第四和第五个筹码移动到位置二的代价都是 1，所以最小总代价为 2。
```

提示：

```
1 <= chips.length <= 100
1 <= chips[i] <= 10^9
```

#### 解决代码

```
// 思路:
// 偶数/奇数移动无需代价,只需要判断偶数和奇数的个数,取小的个数即可
// 奇数少,可以移动奇数的个数,即花费代价.
func MinCostToMoveChips(chips []int) int {
	odd, even := 0,0
	lc := len(chips)
	for i:=0;i<lc;i++ {
		if chips[i] & 0x01 == 0 { //偶数
			even++
		} else {
			odd++
		}
	}
	return min(odd, even)
}

func min(nums ...int) int {
	m := nums[0]
	for i := 1; i < len(nums); i++ {
		if m > nums[i] {
			m = nums[i]
		}
	}
	return m
}
```

#### 测试

```
import "testing"

func TestMinCostToMoveChips(t *testing.T) {
	type args struct {
		chips []int
	}
	tests := []struct {
		name string
		args args
		want int
	}{
		{"test01",args{[]int{1,2,3}},1},
		{"test02",args{[]int{2,2,2,3,3}},2},
	}
	for _, tt := range tests {
		t.Run(tt.name, func(t *testing.T) {
			if got := MinCostToMoveChips(tt.args.chips); got != tt.want {
				t.Errorf("MinCostToMoveChips() = %v, want %v", got, tt.want)
			}
		})
	}
}

```

### 476. 数字的补数

给定一个正整数，输出它的补数。补数是对该数的二进制表示取反。

注意:

给定的整数保证在32位带符号整数的范围内。
你可以假定二进制数不包含前导零位。
示例 1:

```
输入: 5
输出: 2
解释: 5的二进制表示为101（没有前导零位），其补数为010。所以你需要输出2。
```
示例 2:

```
输入: 1
输出: 0
解释: 1的二进制表示为1（没有前导零位），其补数为0。所以你需要输出0。
```

#### 解决

```

package leetcode

// 将num的每一位进行异或,再进行组合,形成新的数.

func FindComplement(num int) int {
	var c, tmp int
	var i uint
	for num != 0 {
		tmp = (num & 0x01) ^ 0x01//取num当前位的异或
		c += tmp<<i  //将num当前位左移i位后
		num >>= 1
		i++
	}
	return c
}
```

#### 测试用例

```
/*
Copyright 2019 louis.
@Time : 2019/10/7 23:51
@Author : louis
@File : 476-numbercomplement
@Software: GoLand

*/

package leetcode

import "testing"

func TestFindComplement(t *testing.T) {
	type args struct {
		num int
	}
	tests := []struct {
		name string
		args args
		want int
	}{
		{"test01",args{5},2},
		{"test02",args{1},0},
		{"test03",args{12},3},
	}
	for _, tt := range tests {
		t.Run(tt.name, func(t *testing.T) {
			if got := FindComplement(tt.args.num); got != tt.want {
				t.Errorf("FindComplement() = %v, want %v", got, tt.want)
			}
		})
	}
}

```

### 318. 最大单词长度乘积

给定一个字符串数组 `words`，找到 `length(word[i]) * length(word[j])` 的最大值，并且这两个单词不含有公共字母。你可以认为每个单词只包含小写字母。如果不存在这样的两个单词，返回 0。

```
示例 1:

输入: ["abcw","baz","foo","bar","xtfn","abcdef"]
输出: 16 
解释: 这两个单词为 "abcw", "xtfn"。
示例 2:

输入: ["a","ab","abc","d","cd","bcd","abcd"]
输出: 4 
解释: 这两个单词为 "ab", "cd"。
示例 3:

输入: ["a","aa","aaa","aaaa"]
输出: 0 
解释: 不存在这样的两个单词。
```
####  解决

```
/*
Copyright 2019 louis.
@Time : 2019/10/8 0:33
@Author : louis
@File : 318-maxproduct
@Software: GoLand

*/

/*
思路:
用二进制的一位表示某一个字母是否出现过，0表示没出现，1表示出现。
"abcd"二进制表示00000000 00000000 00000000 00001111,
"bc"二进制表示00000000 00000000 00000000 00000110。
当两个字符串没有相同的字母时，二进制数与的结果为0。
*/

package leetcode

func String2int(str string) (res int) {
	for i := 0; i < len(str); i++ {  //不能用for-range迭代.
		res |= 1 << uint(str[i]-'a')   // "abc" ==> 二进制"111"
	}
	return res
}

func MaxProduct(words []string) int {
	var arr []int = make([]int, len(words))
	l := len(words)
	for i := 0; i < l; i++ {
		arr[i] = String2int(words[i])  //将[]string数组里面的string,转化为int.
	}
	var res int
	for i := 0; i < l; i++ {
		for j := i + 1; j < l; j++ {
			// 遍历数组,如果数组里面&操作为0,即这两个单词不含有公共字母
			// 并且res < length(word[i]) * length(word[j])时,更新res.
			if arr[i]&arr[j] == 0 && len(words[i])*len(words[j]) > res {
				res = len(words[i]) * len(words[j])
			}
		}
	}
	return res
}

```

#### 测试用例

```
/*
Copyright 2019 louis.
@Time : 2019/10/8 0:33
@Author : louis
@File : 318-maxproduct
@Software: GoLand

*/

package leetcode

import (
	"testing"
)

func TestMaxProduct(t *testing.T) {
	type args struct {
		words []string
	}
	tests := []struct {
		name string
		args args
		want int
	}{
		{"test01", args{[]string{"a", "ab", "abc", "d", "cd", "bcd", "abcd"}}, 4},
		{"test02", args{[]string{"abcw","baz","foo","bar","xtfn","abcdef"}}, 16},
		{"test03", args{[]string{"a","aa","aaa","aaaa"}}, 0},
	}
	for _, tt := range tests {
		t.Run(tt.name, func(t *testing.T) {
			if got := MaxProduct(tt.args.words); got != tt.want {
				t.Errorf("MaxProduct() = %v, want %v", got, tt.want)
			}
		})
	}
}

func TestString2int(t *testing.T) {
	type args struct {
		str string
	}
	tests := []struct {
		name    string
		args    args
		wantRes int
	}{
	//"abcd"二进制表示00000000 00000000 00000000 00001111,
	//"bc"二进制表示00000000 00000000 00000000 00000110。
		{"test01",args{"abcd"},0x0f},  
		{"test02",args{"bc"},0x06},  
	}
	for _, tt := range tests {
		t.Run(tt.name, func(t *testing.T) {
			if gotRes := String2int(tt.args.str); gotRes != tt.wantRes {
				t.Errorf("String2int() = %v, want %v", gotRes, tt.wantRes)
			}
		})
	}
}

```

## 参考

- [位运算](https://studygolang.com/articles/14276)


