---
title: "go马踏棋盘贪心算法"
date: 2019-09-26 17:40:10
tags:
  - 贪心算法
  - algorithm
categories: [algorithm] 
---

[TOC]

## 问题描述：

所谓“马踏棋盘”问题，就是指在中国象棋的棋盘上，用马的走法走遍整个棋盘，在8*8的方格中，每个格都要遍历，且只能遍历一次。

我们把棋盘抽象成一个二维数据，输入起始位置的坐标(x,y),根据马的“日”字走法，将马走的步数写入二维数组，然后输出.

### 解决思路

设当前马的坐标为(x,y)，则下一步可以走的有8 个方向

`(x-2,y+1)、(x-1,y+2)、(x+1,y+2)、(x+2,y+1)、(x+2,y-1)、(x+1,y-2)、(x-1,y-2)、(x-2,y-1)`

创建一个二维数组记录马可以走的8个方向

```
chess     [8][8]int //初始化棋盘
direction = [2][9]int{{0, -2, -1, 1, 2, 2, 1, -1, -2},
		{0, 1, 2, 2, 1, -1, -2, -2, -1}} // 马可以走的8个方向
		探测下一步需要走,可以用下面公式
		x = x+direction[0][i]
		y = y+direction[1][i]
```

每一个马都有八个下一步的选择，我们在满足要求的点中任意找一个进行遍历，当八个点都不满足要求时，就回溯的上一步，找其他点进行遍历。

```
Feasible 该点是不是可以走,超出棋盘界限或者已经走过(棋盘初始为0值),都不能走.
0 <= x && x < 8 && 0 <= y && y < 8 && chess[x][y] == 0
```

## 贪心算法

- 初始化权值; 对每个点进行探索,若方向可以行,则`weight[i][j]++`

```
[ 2  3  4  4  4  4  3  2]
[ 3  4  6  6  6  6  4  3]
[ 4  6  8  8  8  8  6  4]
[ 4  6  8  8  8  8  6  4]
[ 4  6  8  8  8  8  6  4]
[ 4  6  8  8  8  8  6  4]
[ 3  4  6  6  6  6  4  3]
[ 2  3  4  4  4  4  3  2]
```

- `setWeight`占位操作

当(x,y)点被占用的时候,当前节点权值设置为9,位置(x,y)周围所有的可行点权值减1

```
setWeight(2,7)
[ 2  3  4  4  4  4  2  2]  //3-->2
[ 3  4  6  6  6  5  4  3]  //6-->5
[ 4  6  8  8  8  8  6  9]  //4-->9
[ 4  6  8  8  8  7  6  4]  //8-->7
[ 4  6  8  8  8  8  5  4]  //6-->5
[ 4  6  8  8  8  8  6  4]
[ 3  4  6  6  6  6  4  3]
[ 2  3  4  4  4  4  3  2]
```

- `UnsetWeight` 回退操作

需要重新计算`weight[i][j]`的权值, 依次探测周围,若可行,则`weight[i][j]++`; 

其周围可行点的权值`weight[x][y]++`

- 最优路线贪心策略

`NextDirection` 每次走下一步,选择下一步最少的权值,进行贪心算法.

如果不先遍历它的话以后可能会很难遍历到它,即使能遍历到,也需要花费大量的回退操作.

### go代码

```go
/*
Copyright 2019 louis.
@Time : 2019/9/25 22:30
@Author : louis
@File : mataqipan
@Software: GoLand

*/

package main

import (
	"fmt"
	"gogs.wangke.co/go/algo/stack"
)

var (
	chess     [8][8]int //初始化棋盘
	direction = [2][9]int{{0, -2, -1, 1, 2, 2, 1, -1, -2},
		{0, 1, 2, 2, 1, -1, -2, -2, -1}} // 马可以走的8个方向
	cur, next Spot        //当前步数和下一步
	s         stack.Stack //栈
	weight    [8][8]int   //表示该位置周围可行点的数目,比如weight[0][0] = 2 只有两步可走.
)

//Spot 保存当前点的位置及可行走方向是否走过
type Spot struct {
	x int    //行
	y int    //列
	d [9]int //d[i]记录了第i号方向是否已经走过,1表示走过
}
//Feasible 该点是不是可以走,超出棋盘界限或者已经走过,都不能走.
func Feasible(x, y int) bool {
	if 0 <= x && x < 8 && 0 <= y && y < 8 && chess[x][y] == 0 {
		return true
	}
	return false
}


//NextDirection 每次走下一步,选择下一步最少的权值,进行贪心算法,返回下一步的方向
func NextDirection(c Spot) int {
	var MinDirection, Min int
	var x, y int
	Min = 9
	for i := 1; i <= 8; i++ {
		//访问过则不考虑
		if c.d[i] != 0 {
			continue
		}
		x = c.x + direction[0][i]
		y = c.y + direction[1][i]
		//选择最小的权值
		if Feasible(x, y) && weight[x][y] < Min {
			Min = weight[x][y]
			MinDirection = i
		}
	}
	return MinDirection
}


// InitWeight 初始化每个点的权值
// 初始为0; 对每个点进行探索,若方向可以行,则weight[i][j]++
func InitWeight() {
	for i := 0; i < 8; i++ {
		for j := 0; j < 8; j++ {
			for k := 1; k <= 8; k++ {
				x := i + direction[0][k]
				y := j + direction[1][k]
				if Feasible(x, y) {
					weight[i][j]++
				}
			}
		}
	}
}

//SetWeight 当(x,y)点被占用的时候,当前节点权值设置为9,位置(x,y)周围所有的可行点权值减1
func SetWeight(x, y int) {
	for k := 1; k <= 8; k++ {
		weight[x][y] = 9
		i := x + direction[0][k]
		j := y + direction[1][k]
		if Feasible(i, j) {
			weight[i][j]--
		}
	}
}

// UnsetWeight 回退操作,需要重新计算weight[i][j]的权值,
// 依次探测周围,若可行,则weight[i][j]++; 其周围可行点的权值+1
func UnsetWeight(x, y int) {
	for k := 1; k <= 8; k++ {
		weight[x][y] = 0
		i := x + direction[0][k]
		j := y + direction[1][k]
		if Feasible(i, j) {
			weight[x][y]++
			weight[i][j]++
		}
	}
}
//output 输出棋盘
func output() {
	for j := 0; j < 8; j++ {
		fmt.Printf("%2d\n", chess[j])
	}
	//for j := 0; j < 8; j++ {
	//	fmt.Printf("%2d\n", weight[j])
	//}
}

func outWeight() {
	for j := 0; j < 8; j++ {
		fmt.Printf("%2d\n", weight[j])
	}
}

// 当找不到下一个位置时,即NextDirection返回值为0,要进行回退
// 为了回退方便,使用栈来存储, 能进时,当前的位置入栈, 向i走一步
// 回退操作, 在棋牌的cur点置0,Step--; 出栈一个点,设置为当前的cur
// 回退操作, 不能重复探测,去重操作.
func main() {
	InitWeight()
	//outWeight()
	fmt.Scanln(&cur.x,&cur.y)
	backup := 0
	Step := 1
	SetWeight(cur.x, cur.y)
	chess[cur.x][cur.y] = Step
	for Step < 64 {
        //获取下一步访问方向,根据贪心策略,会选择这一步的weight值最少的那个方向
		k := NextDirection(cur)
		if k != 0 {
            //这一步可以走,将这一步记录下来
			next.x = cur.x + direction[0][k]
			next.y = cur.y + direction[1][k]
			cur.d[k] = 1
			s.Push(cur)
			cur = next
			Step++
			chess[cur.x][cur.y] = Step
			SetWeight(cur.x, cur.y)
			//回退
		} else {
            //这步不可以走,得回退到上一步
			chess[cur.x][cur.y] = 0
			backup++//回退次数
			Step--
			UnsetWeight(cur.x, cur.y)
			cur = s.Pop().(Spot)
		}
	}
	output()
	fmt.Print(backup)
}

```

### 依赖的stack

```go
/*
@Time : 2019/9/23 23:29
@Author : louis
@File : stack
@Software: GoLand
*/

package stack

type Item struct {
	item interface{}
	next *Item
}

// Stack is a base structure for LIFO
type Stack struct {
	sp    *Item
	depth uint64
}

// Initialzes new Stack
func New() *Stack {
	var stack = new(Stack)

	stack.depth = 0
	return stack
}

// Pushes a given item into Stack
func (stack *Stack) Push(item interface{}) {
	stack.sp = &Item{item: item, next: stack.sp}
	stack.depth++
}

// Deletes top of a stack and return it
func (stack *Stack) Pop() interface{} {
	if stack.depth > 0 {
		item := stack.sp.item
		stack.sp = stack.sp.next
		stack.depth--
		return item
	}

	return nil
}

// Peek returns top of a stack without deletion
func (stack *Stack) Peek() interface{} {
	if stack.depth > 0 {
		return stack.sp.item
	}

	return nil
}

//IsEmpty returns true means Empty Stack
func (stack *Stack) IsEmpty() bool  {
	return stack.depth == 0
}
```

验证

```bash
$ go run main.go
2 4  
[25 12 27 38 23  2 59 52]
[28 39 24 13 58 51 22  3]
[11 26 37 42  1 60 53 62]
[40 29 14 57 50 63  4 21]
[15 10 41 36 43 56 61 54]
[32 35 30 49 64 47 20  5]
[ 9 16 33 44  7 18 55 46]
[34 31  8 17 48 45  6 19]
0
```

感谢您的阅读~

## 参考

- mooc大学[数据结构与算法](http://www.icourse163.org/course/XIYOU-1002578005)