---
title: "go房贷计算器"
date: 2019-09-24 17:40
tag:
  - 房贷
  - algorithm
---

[TOC]

# 前言

房贷，也被称为房屋抵押贷款.

[房屋贷款](https://baike.baidu.com/item/房屋贷款)的方式有三种，分别是银行[商业贷款](https://baike.baidu.com/item/商业贷款)、[公积金贷款](https://baike.baidu.com/item/公积金贷款)、[组合贷款](https://baike.baidu.com/item/组合贷款)。

总想质疑一下银行的贷款计算器是不是靠谱.故而自己动手写了一个小而简单的计算器,几十行代码

### 等额本息

等额本息还款法即把[按揭贷款](https://baike.baidu.com/item/按揭贷款/2951077)的[本金](https://baike.baidu.com/item/本金/11025685)总额与利息总额相加，然后平均分摊到还款期限的每个月中，每个月的还款额是固定的，但每月还款额中的本金比重逐月递增、利息比重逐月递减。

<script type="text/javascript" async
  src="https://cdnjs.cloudflare.com/ajax/libs/mathjax/2.7.5/MathJax.js?config=default">
</script>

- 原理:

**每个月总偿还金额是不变的.**
$$
设:A_n为第n个月欠银行贷款. X为每个月还款金额(本金+利息).月利率S
$$

$$
故: A_1 = AS(1+S)-X
$$

$$
所以: A_2 = A_1(1+S)-X = A(1+S)^{2}-X[(1+S)+1]
$$

$$
递推得到: A_n = A_{n-1}(1+S)-X
$$
$$
展开得: A_n= A(1+S)^n-X[(1+S)^{n-1}+(1+S)^{n-2}+\cdots+(1+S)+1]
$$
$$
利用等比数列求和公式:A_n= A(1+S)^n-\frac{X[1+S]^n-1}{S}
$$

$$
因此: X = \frac{AS(1+S)^{m}}{(1+S)^m-1}
$$

### 等额本金

等额本金是指一种[贷款](https://baike.baidu.com/item/贷款/1129285)的还款方式，是在还款期内把贷款数总额等分，每月偿还同等数额的本金和剩余贷款在该月所产生的利息，这样由于每月的还款[本金](https://baike.baidu.com/item/本金/11025685)额固定，而利息越来越少，借款人起初还款压力较大，但是随时间的推移每月还款数也越来越少。

- 原理:

**每个月偿还本金都是一样的**,只是偿还的利息越来越少.
$$
设总贷款金额为A,还款月数为Month,月利率为S.第n月还款数为:A_n
$$
$$
故:
A_n = \frac{A}{Month} + (A - \frac{A}{Month} * n) * S
$$

## 代码实现

```
/*
@Time : 2019/9/24 16:48
@Author : louis
@File : paymydebt
@Software: GoLand
*/

package main

import (
	"fmt"
	"math"
)


const (
	bases  float64 = 0.049           //商业贷款基准年利率
	baseS          = bases * 1.1     //上浮10%
	baseG          = 0.032           //公积金贷款基准年利率
	S              = baseS / 12      //商业贷款月利率上浮 10%
	G              = baseG / 12      //公积金贷款月利率
	Start          = 10 - offset     //起始还贷日期如:10月份
	offset         = 1               //偏移量.%12后为0-11.
	Year           = 30              //贷款年份
	Month          = 12*Year + Start //贷款月数12*30
	AG             = 150000          //公积金贷款总额
	AS             = 880000          //商业贷款总额
	//S               = 0.0044916 //商业贷款月利率上浮 10%
)


/*
- 原理:

**每个月偿还本金都是一样的**,只是偿还的利息越来越少.

*/
func benJ() {
	const (
		bs = AS / (Month - Start) //商贷每月偿还本金
		bg = AG / (Month - Start) //公积金每月偿还本金
	)
	var (
		jG               = [Month]float64{} //公积金每月待还
		jS               = [Month]float64{} //商业每月待还
		money            = [Month]float64{} //总待还
		payBackG float64 = 0                //公积金已偿还
		payBacks float64 = 0                //商贷已偿还
		payBack  float64 = 0                //总偿还

	)
	for i := Start; i < Month; i++ {
		jG[i] = bg + (AG-bg*(float64(i-Start)))*G
		payBackG += jG[i]
		jS[i] = bs + (AS-bs*(float64(i-Start)))*S
		payBacks += jS[i]
		money[i] = jG[i] + jS[i]
		payBack = payBacks + payBackG
		//偏移量补足即可
		fmt.Printf("%4d年%2d月还款: %.2f = %.2f[公积金] + %.2f[商贷],已还%.2f\n",
			2019+i/12, i%12+offset, money[i], jG[i], jS[i], payBack)
	}
	fmt.Printf("等额本金共计偿还利息为:%.2f\n", payBack-AS-AG)
}

/*

- 原理:
**每个月总偿还金额是不变的.**
*/
func benX() {
	var es, eg float64
	es = AS * S * math.Pow(1+S, Month) / (math.Pow(1+S, Month) - 1)
	eg = AG * G * math.Pow(1+G, Month) / (math.Pow(1+G, Month) - 1)
	fmt.Printf("%.2f+%.2f = %0.2f\n", eg, es, es+eg)
	fmt.Printf("共计偿还利息为:%.2f", (es+eg)*Month-AG-AS)
}

func main() {
	benJ()
	fmt.Printf("\n等额本息:")
	benX()
}
```

验证

```go
$ go run main.go
2019年10月还款: 7212.67 = 816.00[公积金] + 6396.67[商贷],已还7212.67
2019年11月还款: 7200.58 = 814.89[公积金] + 6385.69[商贷],已还14413.25
2019年12月还款: 7188.49 = 813.78[公积金] + 6374.71[商贷],已还21601.74
...
2049年 8月还款: 2885.53 = 418.86[公积金] + 2466.67[商贷],已还1812626.77
2049年 9月还款: 2873.45 = 417.75[公积金] + 2455.70[商贷],已还1815500.21
等额本金共计偿还利息为:785500.21

等额本息:639.29+4887.91 = 5527.20
共计偿还利息为:1009535.42
```

感谢阅读~~

### 参考

- [等额本金](https://baike.baidu.com/item/等额本金)

- [等额本息](https://baike.baidu.com/item/等额本息)