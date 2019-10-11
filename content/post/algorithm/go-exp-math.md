---
title: "Go 运算表达式求值"
date: 2019-09-13 11:40:10
tags: 
  - stack
  - algorithm
  - exp
categories: [algorithm] 
---

[TOC]

> 运算表达式参考学习韩老师的golang教程.

## 设计思路

采用`golang`实现常用的表达式求值,不包含括号,这边暂时没有实现.

数据结构使用Stack如下,使用两个栈分别来存储操作数和运算符.

```go
const MaxTop = 30
type Stack struct {
	Top int
	arr [MaxTop]int
}
```

思考需要解决哪些问题才能解决表达式求值呢?

- 如何读取表达式的运算符和操作数.
- 操作符和运算数如何进行比较
- 如何使得操作数每次运算都能安装优先来进行运算

为了解决上面的三个问题,思考如下.

```
// 1. 设计一个算法,读取表达式中的每个字符.对表达式进行切分,我们知道string类型底层实现为slice切片.
每取一个字符,对index++,循环读取即可完成对每个操作运算符的提取.

ch := exp[index : index+1] // "3" 单个字符串, "+" ==> 43

// 2. 将单个字符串转为ASCII对应的十进制数. 

temp := int([]byte(ch)[0]) // 字符串转为byte,  

/* 3. 对取出来的每个ASCII的十进制数进行判断,如果为操作数,则直接入操作数栈
如果为运算符,则需要进行判断:
a. 如果符号栈为空,则符号栈直接入栈.
b. 如果符号栈不为空,则进行比对,如果
	A. 当前符号栈顶的运算符优先级 >= 要入栈的运算符优先级, 从符号栈弹出一个运算符,从操作数栈弹出两个数,进行运算,运算结果存入操作数栈. (这里需要循环比对,如果不循环比对,+ * - * -,最后没有办法求解)
	B. 当前符号栈顶的运算符优先级 < 要入栈的运算符优先级,则直接入符号栈.
4. 执行这些后,当前操作数栈和符号栈的栈底到栈顶的优先级,始终是从低到高,每次弹出2个操作数,一个运算符,进行计算后,将结果压入操作数栈,循环操作,最后一个存入的操作数即为结果.
*/

这里有个漏洞,如果是多位数操作怎么办,这样的思路就错了,每次读入一位数,最后的结果肯定不是我们所想.
```

## 进行运算和判断的方法

```go
// 1.判断Stack为空方法

func (s *Stack) IsEmpty() bool {
	return s.Top == -1 
}

// 2. 判断Stack已满方法
func (s *Stack) IsFull() bool {
	return s.Top == MaxTop-1
}

// 3. 判断Stack的大小方法
func (s *Stack) Size() int {
	return len(arr[:s.Top])
}

// 4. 入栈
func (s *Stack) Push(val int) bool {
	if s.IsFull() {
		return false
	}
	s.Top++
	s.arr[s.Top]= val
	return true
}

// 5. 出栈
func (s *Stack) Pop()(val int, bool) {
	if s.IsEmpty() {
		returnn -1, false
	}
	val = s.arr[s.Top]
	s.Top--
	return val, true
}
// 6. 判断是否为运算符
func (s *Stack) IsOpr(val int) bool {
	if val == 42 || val == 43 || val == 45 ||
		val == 47 || val == 94 {
		return true
	}	
	return false
}
// 7.0 计算幂(整形)
func power(a, n int) int {
	res := 1
	if n != 0 {
		res *= a
		n--
	}
	retuen res
}
// 7. 计算操作数与运算符
func (s *Stack) Cal(a, b, opr int) (res int) {
	switch opr {
	case 42: 
		res = b * a
	case 43: 
		res = b + a
	case 45:
		res = b - a
	case 47:
		res = b / a
	case 94:
		res = power(b, a) // b^a
	}
	return res
}

// 8. 定义运算符的优先级
func (s *Stack) Nice(opr int) (res int) {
	switch {
	case opr == 43 || opr == 45:  // * / 
		res = 1 
	case opr == 94:   // ^
		res = 2
	case opr == 42 || opr == 47: // + -
		fallthough	
    default:   
        res = 0
	}
	return res
}
```

## Exp求值

如果给出的exp符合表达式,那么结果一定存在.

```go
func Exp(exp string) (res int) {
		numStack := &Stack{
		Top: -1,
	}
	oprStack := &Stack{
		Top: -1,
	}
	index, a, b, opr, res := 0, 0, 0, 0, 0
	keepNum := ""
	for {
		ch := exp[index : index+1] // "3" 单个字符串, "+" ==> 43
		fmt.Println(ch)
		temp := int([]byte(ch)[0]) //字符串转为byte,  字符转的ASCII码
		if oprStack.IsOpr(temp) {  // 如果是数字,则进入数字处理逻辑.
			// 如果operStack是空栈,直接入栈;
			// 并将数栈也pop出两个数,进行运算,
			// 将运算的结果push到数栈,符号再入符号栈
			if oprStack.IsEmpty() {
				oprStack.Push(temp)
			} else {
				//不是空栈的话,如果栈顶的运算符优先级,
				//大于当前准备入栈的运算符优先级,先pop出栈
				//例如 栈顶运算符为 * ,准备 入栈的运算符为 + ,
				//则先出栈. 并从操作栈取出两个操作数进行运算,
				//再把结果压入操作栈==>
				//继续进行比较, 如果栈顶的运算符优先级大于
				//当前准备入栈的运算符优先级,先pop出栈.
                //直到栈为空.或者栈的优先级从低到高排列.
				for oprStack.Nice(oprStack.arr[oprStack.Top]) >=
					oprStack.Nice(temp) {
					a, _ = numStack.Pop()
					b, _ = numStack.Pop()
					opr, _ = oprStack.Pop()
					res = oprStack.Cal(a, b, opr)
					//运算结果重新入数栈
					numStack.Push(res)
                    // 弹出opr运算符之后,进行判空处理,为空就直接跳出循环
                    // 直接将待入栈的运算符压入符号栈.
					if oprStack.IsEmpty() {
						break
					}
				}
				oprStack.Push(temp)
			}

		} else { //数字,如何处理多位数的逻辑.
			//处理多位数 keepNum string,拼接.
			keepNum += ch
			if index == len(exp)-1 {
                // 如果是最后的一个数,则直接将str '3' ==> 转换为int 3
				val, _ := strconv.ParseInt(keepNum, 10, 64)
				numStack.Push(int(val)) // 3 ==> 51(ASCII)
			} else {
				if oprStack.IsOpr(int([]byte(exp[index+1 : index+2])[0])) {
                    // 如果下一个字符是运算符,则直将keepNum压入操作数栈.
                    // 否则就一直进行keepNum += ch 拼接操作.
					val, _ := strconv.ParseInt(keepNum, 10, 64)
					numStack.Push(int(val)) // 3 ==> 51(ASCII)
                    // 操作数入栈后,keep置空.
					keepNum = ""
				}
			}
		}
		// 判断index是否已经扫完
		if index+1 == len(exp) {
			break
		}
		index++
	}

	// 优先级高的已经计算完,或者优先级从低到高在符号栈排列.

	for { //符号栈为空就跳出循环,这时候操作数栈最后的一个数肯定就是exp的结果.
		if oprStack.IsEmpty() {
			break
		}
		a, _ = numStack.Pop()
		b, _ = numStack.Pop()
		opr, _ = oprStack.Pop()
		res = oprStack.Cal(a, b, opr)
		//运算结果重新入数栈
		numStack.Push(res)
	}

	res, _ = numStack.Pop()
	return res
}
```

## 验证

给出一个表达式,来进行验证.

```go
func main() {
	exp := "30+30*6-4^2-6"
	res := Exp(exp)
	fmt.Printf("exp %s = %v", exp, res)
}
/*output
exp 30+30*6-4^2-6 = 188
*/
```

- 整体的代码

```
package main

import (
	"fmt"
	"strconv"
)

const MaxTop = 30

type Stack struct {
	Top int
	arr [MaxTop]int
}

func (s *Stack) IsEmpty() bool {
	return s.Top == -1
}

func (s *Stack) IsFull() bool {
	return s.Top == MaxTop-1
}

func (s *Stack) Size() int {
	return len(s.arr[:s.Top])
}

func (s *Stack) Push(val int) (b bool) {
	if s.IsFull() {
		return false
	}
	s.Top++
	s.arr[s.Top] = val
	fmt.Printf("stack push  %#v\n", val)
	return true
}

func (s *Stack) Pop() (val int, b bool) {
	if s.IsEmpty() {
		return 0,false
	}
	val = s.arr[s.Top]
	s.Top--
	fmt.Printf("stach pop %#v\n", val)
	return val, true
}

func (s *Stack) List() {
	if s.IsEmpty() {
		return
	}
	//for k,v :=range s.arr {
	//	defer fmt.Printf("arr[%d] = %d\n",k,v)
	//}
	fmt.Println("Stack -->")
	for i := s.Top; i >= 0; i-- {
		fmt.Printf("arr[%d] = %d\n", i, s.arr[i])
	}
}

// 判断是否为运算符[+,-,*,/,^]
func (s *Stack) IsOpr(val int) bool {
	/* ASCII '42 * 43 + 45 - 47 /' */
	if val == 42 || val == 43 || val == 45 || val == 47 || val == 94 {
		return true
	} else {
		return false
	}

}

func Power(a, n int) int {
	res := 1
	for n != 0 {
		res *= a
		n--
	}
	return res
}

func (s *Stack) Cal(a, b, opr int) int {
	res := 0
	switch opr {
	case 42:
		res = b * a
	case 43:
		res = b + a
	case 45:
		res = b - a
	case 47:
		res = b / a
	case 94:
		res = Power(b, a) // b^a
	default:
		fmt.Println("opr is err")
	}
	return res
}

// 编写方法,返回运算符的优先级[自定义]

func (s *Stack) Nice(opr int) int {
	/* * / Nice返回1
	   + - Nice返回0	
	   ^   Nice返回2*/
	res := 0
	if opr == 42 || opr == 47 {
		return 1
	} else if opr == 43 || opr == 45 {
		return 0
	} else if opr == 94 { // 94	^
		return 2
	}
	return res
}

func Exp(exp string) (res int) {
	numStack := &Stack{
		Top: -1,
	}
	oprStack := &Stack{
		Top: -1,
	}
	index, a, b, opr, res := 0, 0, 0, 0, 0
	keepNum := ""
	for {
		ch := exp[index : index+1] // "3" 单个字符串, "+" ==> 43
		temp := int([]byte(ch)[0]) //字符串转为byte,  字符转的ASCII码
		if oprStack.IsOpr(temp) {  // 如果是数字,则进入数字处理逻辑.
			// 如果operStack是空栈,直接入栈;
			// 并将数栈也pop出两个数,进行运算,
			// 将运算的结果push到数栈,符号再入符号栈
			if oprStack.IsEmpty() {
				oprStack.Push(temp)
			} else {
				//不是空栈的话,如果栈顶的运算符优先级,
				//大于当前准备入栈的运算符优先级,先pop出栈
				//例如 栈顶运算符为 * ,准备 入栈的运算符为 + ,
				//则先出栈. 并从操作栈取出两个操作数进行运算,
				//再把结果压入操作栈==>
				//继续进行比较, 如果栈顶的运算符优先级大于
				//当前准备入栈的运算符优先级,先pop出栈.
                //直到栈为空.或者栈的优先级从低到高排列.
				for oprStack.Nice(oprStack.arr[oprStack.Top]) >=
					oprStack.Nice(temp) {
					a, _ = numStack.Pop()
					b, _ = numStack.Pop()
					opr, _ = oprStack.Pop()
					res = oprStack.Cal(a, b, opr)
					//运算结果重新入数栈
					numStack.Push(res)
                    // 弹出opr运算符之后,进行判空处理,为空就直接跳出循环
                    // 直接将待入栈的运算符压入符号栈.
					if oprStack.IsEmpty() {
						break
					}
				}
				oprStack.Push(temp)
			}

		} else { //数字,如何处理多位数的逻辑.
			//处理多位数 keepNum string,拼接.
			keepNum += ch
			if index == len(exp)-1 {
                // 如果是最后的一个数,则直接将str '3' ==> 转换为int 3
				val, _ := strconv.ParseInt(keepNum, 10, 64)
				numStack.Push(int(val)) // 3 ==> 51(ASCII)
			} else {
				if oprStack.IsOpr(int([]byte(exp[index+1 : index+2])[0])) {
                    // 如果下一个字符是运算符,则直将keepNum压入操作数栈.
                    // 否则就一直进行keepNum += ch 拼接操作.
					val, _ := strconv.ParseInt(keepNum, 10, 64)
					numStack.Push(int(val)) // 3 ==> 51(ASCII)
                    // 操作数入栈后,keep置空.
					keepNum = ""
				}
			}
		}
		// 判断index是否已经扫完
		if index+1 == len(exp) {
			break
		}
		// 每次扫完都必须index++,扫描下一个exp
		index++
	}

	// 优先级高的已经计算完,或者优先级从低到高在符号栈排列.

	for { //符号栈为空就跳出循环,这时候操作数栈最后的一个数肯定就是exp的结果.
		if oprStack.IsEmpty() {
			break
		}
		a, _ = numStack.Pop()
		b, _ = numStack.Pop()
		opr, _ = oprStack.Pop()
		res = oprStack.Cal(a, b, opr)
		//运算结果重新入数栈
		numStack.Push(res)
	}

	res, _ = numStack.Pop()
	return res
}

func main() {
	exp := "30+30*6-4^2-6"
	res := Exp(exp)
	fmt.Printf("exp %s = %v", exp, res)
}

//stack push  30
//stack push  43
//stack push  30
//stack push  42
//stack push  6
//stack push  180
//stack push  210
//stack push  45
//stack push  4
//stack push  94
//stack push  2
//stack push  16
//stack push  194
//stack push  45
//stack push  6
//stack push  188
//exp 30+30*6-4^2-6 = 188

```

### 参考

- 韩顺平`golang`教程.
- [数据结构与算法](https://www.icourse163.org/course/XIYOU-1002578005)