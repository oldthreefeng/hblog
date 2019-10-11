---
title: "go Paxos"
date: 2019-08-28 18:10:10
tags: 
  - go
  - Paxos
  - algorithm
categories: [algorithm] 
---

[TOC]

# Paxos

## 问题描述和假设

分布式系统中的节点通信存在两种模型：[共享内存](https://zh.wikipedia.org/wiki/共享内存)（Shared memory）和[消息传递](https://zh.wikipedia.org/wiki/消息传递)（Messages passing）。基于消息传递通信模型的分布式系统，不可避免的会发生以下错误：进程可能会慢、被杀死或者重启，消息可能会延迟、丢失、重复，在基础 Paxos 场景中，先不考虑可能出现消息篡改即[拜占庭错误](https://zh.wikipedia.org/wiki/拜占庭将军问题)的情况。Paxos 算法解决的问题是在一个可能发生上述异常的[分布式系统](https://zh.wikipedia.org/wiki/分布式计算)中如何就某个值达成一致，保证不论发生以上任何异常，都不会破坏决议的一致性。一个典型的场景是，在一个分布式数据库系统中，如果各节点的初始状态一致，每个节点都执行相同的操作序列，那么他们最后能得到一个一致的状态。为保证每个节点执行相同的命令序列，需要在每一条指令上执行一个“一致性算法”以保证每个节点看到的指令一致。一个通用的一致性算法可以应用在许多场景中，是分布式计算中的重要问题。因此从20世纪80年代起对于一致性算法的研究就没有停止过。

为描述Paxos算法，Lamport虚拟了一个叫做Paxos的[希腊城邦](https://zh.wikipedia.org/wiki/希臘城邦)，这个岛按照议会民主制的政治模式制订法律，但是没有人愿意将自己的全部时间和精力放在这种事情上。所以无论是议员，议长或者传递纸条的服务员都不能承诺别人需要时一定会出现，也无法承诺批准决议或者传递消息的时间。但是这里假设没有[拜占庭将军问题](https://zh.wikipedia.org/wiki/拜占庭将军问题)（Byzantine failure，即虽然有可能一个消息被传递了两次，但是绝对不会出现错误的消息）；只要等待足够的时间，消息就会被传到。另外，Paxos岛上的议员是不会反对其他议员提出的决议的。

对应于分布式系统，议员对应于各个节点，制定的法律对应于系统的状态。各个节点需要进入一个一致的状态，例如在独立[Cache](https://zh.wikipedia.org/wiki/Cache)的[对称多处理器](https://zh.wikipedia.org/w/index.php?title=对称多处理器&action=edit&redlink=1)系统中，各个处理器读内存的某个字节时，必须读到同样的一个值，否则系统就违背了一致性的要求。一致性要求对应于法律条文只能有一个版本。议员和服务员的不确定性对应于节点和消息传递通道的不可靠性

### 算法的提出与证明

首先将议员的角色分为 proposers，acceptors，和 learners（允许身兼数职）。proposers 提出提案，提案信息包括提案编号和提议的 value；acceptor 收到提案后可以接受（accept）提案，若提案获得多数派（majority）的 acceptors 的接受，则称该提案被批准（chosen）；learners 只能“学习”被批准的提案。划分角色后，就可以更精确的定义问题：

1. 决议（value）只有在被 proposers 提出后才能被批准（未经批准的决议称为“提案（proposal）”）；
2. 在一次 Paxos 算法的执行实例中，只批准（chosen）一个 value；
3. learners 只能获得被批准（chosen）的 value。

```
在 Leslie Lamport 之后发表的paper中将 majority 替换为更通用的 quorum 概念，但在描述classic paxos的论文  Paxos made simple 中使用的还是majority的概念。
```

另外还需要保证 progress。这一点以后再讨论。

作者通过不断加强上述3个约束（主要是第二个）获得了 Paxos 算法。

批准 value 的过程中，首先 proposers 将 value 发送给 acceptors，之后 acceptors 对 value 进行接受（accept）。为了满足只批准一个 value 的约束，要求经“多数派（majority）”接受的 value 成为正式的决议（称为“批准”决议）。这是因为无论是按照人数还是按照权重划分，两组“多数派”至少有一个公共的 acceptor，如果每个 acceptor 只能接受一个 value，约束2就能保证。

于是产生了一个显而易见的新约束：

```
P1：一个 acceptor 必须接受（accept）第一次收到的提案。
```

注意 P1 是不完备的。如果恰好一半 acceptor 接受的提案具有 value A，另一半接受的提案具有 value B，那么就无法形成多数派，无法批准任何一个 value。

约束2并不要求只批准一个提案，暗示可能存在多个提案。只要提案的 value 是一样的，批准多个提案不违背约束2。于是可以产生约束 P2：

```
P2：一旦一个具有 value v 的提案被批准（chosen），那么之后批准（chosen）的提案必须具有 value v。
```

注：通过某种方法可以为每个提案分配一个编号，在提案之间建立一个全序关系，所谓“之后”都是指所有编号更大的提案。

如果 P1 和 P2 都能够保证，那么约束2就能够保证。

批准一个 value 意味着多个 acceptor 接受（accept）了该 value。因此，可以对 P2 进行加强：

```
P2a：一旦一个具有 value v 的提案被批准（chosen），那么之后任何 acceptor 再次接受（accept）的提案必须具有 value v。
```

由于通信是异步的，P2a 和 P1 会发生冲突。如果一个 value 被批准后，一个 proposer 和一个 acceptor 从休眠中苏醒，前者提出一个具有新的 value 的提案。根据 P1，后者应当接受，根据 P2a，则不应当接受，这种场景下 P2a 和 P1 有矛盾。于是需要换个思路，转而对 proposer 的行为进行约束：

```
P2b：一旦一个具有 value v 的提案被批准（chosen），那么以后任何 proposer 提出的提案必须具有 value v。
```

由于 acceptor 能接受的提案都必须由 proposer 提出，所以 P2b 蕴涵了 P2a，是一个更强的约束。

但是根据 P2b 难以提出实现手段。因此需要进一步加强 P2b。

假设一个编号为 m 的 value v 已经获得批准（chosen），来看看在什么情况下对任何编号为 n（n>m）的提案都含有 value v。因为 m 已经获得批准（chosen），显然存在一个 acceptors 的多数派 C，他们都接受（accept）了v。考虑到任何多数派都和 C 具有至少一个公共成员，可以找到一个蕴涵 P2b 的约束 P2c：

```
P2c：如果一个编号为 n 的提案具有 value v，那么存在一个多数派，要么他们中所有人都没有接受（accept）编号小于 n 
的任何提案，要么他们已经接受（accept）的所有编号小于 n 的提案中编号最大的那个提案具有 value v。
```

可以用[数学归纳法](https://zh.wikipedia.org/wiki/数学归纳法)证明 P2c 蕴涵 P2b：

假设具有value v的提案m获得批准，当n=m+1时，采用反证法，假如提案n不具有value v，而是具有value w，根据P2c，则存在一个多数派S1，要么他们中没有人接受过编号小于n的任何提案，要么他们已经接受的所有编号小于n的提案中编号最大的那个提案是value w。由于S1和通过提案m时的多数派C之间至少有一个公共的acceptor，所以以上两个条件都不成立，导出矛盾从而推翻假设，证明了提案n必须具有value v；

若（m+1）..（N-1）所有提案都具有value v，采用反证法，假如新提案N不具有value v，而是具有value w',根据P2c，则存在一个多数派S2，要么他们没有接受过m..（N-1）中的任何提案，要么他们已经接受的所有编号小于N的提案中编号最大的那个提案是value w'。由于S2和通过m的多数派C之间至少有一个公共的acceptor，所以至少有一个acceptor曾经接受了m，从而也可以推出S2中已接受的所有编号小于n的提案中编号最大的那个提案的编号范围在m..（N-1）之间，而根据初始假设，m..（N-1）之间的所有提案都具有value v，所以S2中已接受的所有编号小于n的提案中编号最大的那个提案肯定具有value v，导出矛盾从而推翻新提案n不具有value v的假设。根据数学归纳法，我们证明了若满足P2c，则P2b一定满足。

P2c是可以通过消息传递模型实现的。另外，引入了P2c后，也解决了前文提到的P1不完备的问题。

## `golang`实现

### 实现的步骤

```
//a. Proposer选择一个提案编号N，然后向半数以上的Acceptor发送编号为N的Prepare请求。
//a.1 生成编号N的算法为 `p.lastSeq <<16 | p.id`

//b. acceptor返回promise,并保证小于n的提案不在接收

//c.1 更新proposer里面的acceptor,保证proposer存入的acceptor提案N为最新
//c.2 acceptor里返回的promise达到半数以上
//c.3 如果Proposer收到半数以上Acceptor对其发出的编号为N的Prepare请求的响应，
//   那么它就会发送一个针对[N,V]提案的Accept请求给半数以上的Acceptor

// d. 接收proposer的提案N,如果已接收提案N>返回的propose的提案N,则忽略
//    如果已接收提案N< 返回的propose提案N,说明错误;违反了P2c原则
//    如果相同,则接收提案N,将propose提案存入accept里面,并将accept.typ改为接受.

//e. 将accept信息发送至learner,通知learner我接受提案N

//f. learner等待acceptor发送accept mes
//f.1 如果消息类型不为Accept, 返回错误
//f.2 从accepted消息中来进行比对,如果接收的提案N > learner存入的提案N,需要重新学习;否则就忽略

//g. 如果半数以上的learner选择了提案N,则说明选择完毕
```

### 代码

```go
/*
@Time : 2019/8/27 18:39
@Author : louis
@File : paxos
@Software: GoLand
*/

package paxos

import (
	"fmt"
	"log"
	"time"
)

const (
	//Prepare 准备发送的消息
	Prepare = iota + 1
	//Propose 待批准的消息-提案消息,并保证小于n的提案不在接收
	Propose
	//Promise accept返回的消息
	Promise
	//Accept 已经接收的提案
	Accept
)

//promise 接口只能获取消息的提案号
type promise interface {
	number() int
}

type accept interface {
	//accept接口获取提案的值
	proposalValue() string
	//accept接口获取提案的号
	proposalNumber() int
}

//mes 消息体
type mes struct {
	from, to int    //消息发送\接收
	typ      int    //消息类型
	n        int    //提案号
	pren     int    //前一个提案号
	value    string //提案值value
}

//proposalValue 只有是accept和promise类型的消息,才具备提案value值
func (m mes) proposalValue() string {
	//返回value需要什么条件呢？
	switch m.typ {
	case Accept, Promise:
		return m.value
	default:
		panic("unexpect proposalValue")
	}

}

func (m mes) proposalNumber() int {
	switch m.typ {
	case Promise:
		return m.pren
	case Accept:
		return m.n
	default:
		panic("unexpect proposalNumber")
	}
}

func (m mes) number() int {
	return m.n
}

//network to send and recv mes
type network interface {
	send(m mes)
	recv(timeout time.Duration) (mes, bool)
}

// paxosNet paxos 的消息管道,recv[i]消息的接收i接收的信息
type paxosNet struct {
	recv map[int]chan mes
}

// newPaxNet 生成paxosNet,根据agent的数量生成
func newPaxNet(agents ...int) *paxosNet {
	pn := &paxosNet{recv: make(map[int]chan mes, 0)}
	for _, a := range agents {
		pn.recv[a] = make(chan mes, 1024)
	}
	return pn
}

//send 发送消息mes至pn中的接收者i
func (pn *paxosNet) send(m mes) {
	log.Printf("nt send message :%+v", m)
	pn.recv[m.to] <- m
}

//rec 从agent接收mes,并输出,返回信息mes和bool.
func (pn *paxosNet) rec(from int, timeout time.Duration) (mes, bool) {
	select {
	case m := <-pn.recv[from]:
		log.Printf("nt recv message :%+v", m)
		return m, true
	case <-time.After(timeout):
		return mes{}, false
	}
}

//agentNet 根据agent的id生成AgentNet结构体
func (pn *paxosNet) agentNet(id int) *agentNet {
	return &agentNet{id: id, pn: pn}
}

//Empty 判断pn是不是空
func (pn *paxosNet) empty() bool {
	var n int
	for i, q := range pn.recv {
		log.Printf("nt %+v left %d", i, len(q))
		n += len(q)
	}
	return n == 0
}

//AgentNet 代理网络结构体,存放代理的id号和pn消息网络结构体
type agentNet struct {
	id int
	pn *paxosNet
}

func (an *agentNet) send(m mes) {
	an.pn.send(m)
}

func (an *agentNet) recv(timeout time.Duration) (mes, bool) {
	return an.pn.rec(an.id, timeout)
}

//proposer 提案提出者
type proposer struct {
	id        int    //提案号
	lastSeq   int    //上一条Seq消息好
	value     string //提案的value
	valueN    int
	acceptors map[int]promise //接受者的消息map
	nt        network //paxos的网络
}

//newPropose 生成新的提案提出者proposer
func newPropose(id int, value string, nt network, acceptors ...int) *proposer {
	p := &proposer{
		id:        id,
		lastSeq:   0,
		nt:        nt,
		value:     value,
		acceptors: make(map[int]promise),
	}
	for _, a := range acceptors { //遍历acceptors,生成proposer
		p.acceptors[a] = mes{}
	}
	return p
}

func (p *proposer) run() {
	var ok bool
	var m mes
	//c.2 acceptor里返回的promise达到半数以上
	for !p.quorumCheck() {
		if !ok {
			//a. Proposer准备prepare,生成提案编号N，然后向半数以上的Acceptor发送编号为N的Prepare请求。
			ms := p.prepare()
			for i := range ms {
				p.nt.send(ms[i])
			}
		}
		m, ok = p.nt.recv(time.Second)
		// 返回数据失败,说明此次的prepare失败,重新prepare
		if !ok {
			continue
		}
		// prepare成功
		switch m.typ {
		case Promise:
			//c.1 更新proposer里面的acceptor,保证proposer存入的acceptor提案N为最新
			p.receivePromise(m)
		default:
			log.Panicf("proposer: %d unexpected message type: %v", p.id, m.typ)
		}
	}
	log.Printf("%d promise %d reached quorum %d", p.id, p.n(), p.quorum())
	//c.3 如果Proposer收到半数以上Acceptor对其发出的编号为N的Prepare请求的响应，
	//   那么它就会发送一个针对[N,V]提案的Accept请求给半数以上的Acceptor
	ms := p.propose()
	for i := range ms {
		fmt.Printf("proposer %d: ", p.id)
		p.nt.send(ms[i])
	}
}

//大多数,半数+1即为大多数
func (p *proposer) quorum() int {
	return len(p.acceptors)/2 + 1
}

//c.2 acceptor返回的promise大于半数以上同意
func (p *proposer) quorumCheck() bool {
	m := 0
	for _, promise := range p.acceptors {
		// promise里面的提案N 必须和 proposer的提案N相同.
		if promise.number() == p.n() {
			m++
		}
	}
	if m >= p.quorum() {
		return true
	}
	return false
}

//在一个paxos实例中，每个提案需要有不同的编号，且编号间要存在全序关系
func (p *proposer) n() int {
	// 把"<<"左边的运算数的各二进位全部左移若干位，由"<<"右边的数指定移动的位数，高位丢弃，低位补0
	// 再把结果和p.id 进行或运算,将1位合并置1
	return p.lastSeq<<16 | p.id
}

//c.3 如果Proposer收到半数以上Acceptor对其发出的编号为N的Prepare请求的响应，把proposer的value携带上.
//   那么它就会发送一个针对[N,V]提案的Accept请求给半数以上的Acceptor
func (p *proposer) propose() []mes {
	m := make([]mes, p.quorum())
	i := 0
	// 取acceptors里面的index,promise,只能取promise里面的mes.n
	for to, promise := range p.acceptors {
		//如果acceptors的n,和proposer的提案n相等,即acceptor接收proposer的提案n.
		if promise.number() == p.n() {
			m[i] = mes{
				from:  p.id,
				to:    to,
				typ:   Propose,
				n:     p.n(),
				value: p.value,
			}
			i++
		}
		if i == p.quorum() {
			break
		}
	}
	return m
}

//a. Proposer选择一个提案编号N，然后向半数以上的Acceptor发送编号为N的Prepare请求。
func (p *proposer) prepare() []mes {
	p.lastSeq++
	m := make([]mes, p.quorum())
	i := 0
	// 只取acceptors里面的index,即acceptors
	for to := range p.acceptors {
		m[i] = mes{
			from: p.id,
			to:   to,
			typ:  Prepare,
			n:    p.n(),
		}
		i++
		if i == p.quorum() {
			break
		}
	}
	return m
}

//c.1 更新proposer里面的acceptor,保证proposer存入的acceptor提案N为最新
// 从acceptor接收的promise消息.
func (p *proposer) receivePromise(promise mes) {
	prePromise := p.acceptors[promise.from]
	// 从acceptor接收的promise和propose里面的number进行比对
	// propose.acceptors[promise.from].number() < promise.number()
	// 返回的promise大,则更新proposer里面的promise为最新版.
	if prePromise.number() < promise.number() {
		log.Printf("proposer: %d received a new promise %+v", p.id, promise)
		p.acceptors[promise.from] = promise
		//acceptors返回的提案preN > proposer的N
		if promise.proposalNumber() > p.valueN {
			log.Printf("proposer: %d updated the value [%s] to %s",
				p.id, p.value, promise.proposalValue())
			//promise.pren = p.n()
			p.valueN = promise.proposalNumber()
			p.value = promise.proposalValue()
		}
	}
}

type acceptor struct {
	id int
	//一旦acceptor接收提案propose;
	// 便需要通知所有learners,通信总次数为 M * N
	//TODO 通知learner集合,再由learner集合通知剩下的learner
	learners []int
	accept   mes
	promised promise
	nt       network
}

func newAcceptor(id int, nt network, learners ...int) *acceptor {
	return &acceptor{
		id:       id,
		nt:       nt,
		promised: mes{},
		learners: learners,
	}
}

func (a *acceptor) run() {
	for {
		m, ok := a.nt.recv(time.Hour)
		if !ok {
			continue
		}
		switch m.typ {
		case Propose:
			//d. 接收proposer的提案N,如果已接收提案N>返回的propose的提案N,则忽略
			//    如果已接收提案N< 返回的propose提案N,说明错误;违反了P2c原则
			//    如果相同,则接收提案N,将propose提案存入accept里面,并将accept.typ改为接受.
			accepted := a.receivePropose(m)
			if accepted {
				//e. 将accept信息发送至learner,通知learner我接受提案N
				for _, l := range a.learners {
					m = a.accept
					m.from = a.id
					m.to = l
					a.nt.send(m)
				}
			}
			//b. acceptor返回promise,并保证小于n的提案不在接收
		case Prepare:
			promised, ok := a.receivePrepare(m)
			if ok {
				a.nt.send(promised)
			}
		default:
			log.Panicf("accepted : %d message tpye unknwon: %d", a.id, m.typ)
		}
	}
}

// d. 接收proposer的提案N,如果已接收提案N>返回的propose的提案N,则忽略
//    如果已接收提案N< 返回的propose提案N,说明错误;违反了P2c原则
//    如果相同,则接收提案N,将propose提案存入accept里面,并将accept.typ改为接受.
//P2c：如果一个编号为 n 的提案具有 value v，那么存在一个多数派，要么他们中所有人都没有接受（accept）编号小于 n
//的任何提案，要么他们已经接受（accept）的所有编号小于 n 的提案中编号最大的那个提案具有 value v。
func (a *acceptor) receivePropose(propose mes) bool {
	// 已接收提案N > propose的mes.n;不接收这个提案
	if a.promised.number() > propose.number() {
		log.Printf("acceptor %d [promised: %+v] ignored propose mes: %+v", a.id, a.promised, propose)
		return false
	}
	//已接收提案N < propose的mes.n;
	if a.promised.number() < propose.number() {
		log.Panicf("acceptor %d [promised: %+v] received unexpected proposal mes: %+v",
			a.id, a.promised, propose)
	}
	log.Printf("acceptor %d [promised: %+v, accept: %+v]  accepted propose: %+v",
		a.id, a.promised, a.accept, propose)
	a.accept = propose
	a.accept.typ = Accept
	return true
}

//b. acceptor返回promise,并保证小于n的提案不在接收
func (a *acceptor) receivePrepare(prepare mes) (promised mes, b bool) {
	// 如果获取的m.n大于提案N,Promised提案接收,承诺不再接收任何小于N的提案
	// P1:一个 acceptor 必须接受（accept）第一次收到的提案。
	if a.promised.number() < prepare.number() {
		log.Printf("acceptor %d [promised: %+v]  promised %+v", a.id, a.promised, prepare)
		a.promised = prepare
		//把消息返回
		promised = mes{
			typ:   Promise,
			from:  a.id,
			to:    prepare.from,
			n:     a.promised.number(),
			pren:  a.accept.n,
			value: a.accept.value,
		}
		return promised, true
	}
	log.Printf("acceptor %d [promised: %+v] ignored prepare mes: %+v", a.id, a.promised, prepare)
	return mes{}, false
}

type learner struct {
	id        int
	acceptors map[int]accept
	nt        network
	value     chan string //测试数据比对,learner学习后得到的提案N对应的V[N,V]
}

func newLearner(id int, nt network, acceptors ...int) *learner {
	l := &learner{id: id, nt: nt, acceptors: make(map[int]accept), value: make(chan string)}
	for _, a := range acceptors {
		l.acceptors[a] = mes{typ: Accept}
	}
	return l
}

func (l *learner) GetValue() (v string) {
	select {
	case v := <-l.value:
		return v
	case <-time.After(time.Second):
		return
	}
}

func (l *learner) learn() {
	for {
		//f. 等待acceptor发送accept mes,
		m, ok := l.nt.recv(time.Hour)
		if !ok {
			continue
		}
		//f.1 如果消息类型不为Accept, 返回错误
		if m.typ != Accept {
			log.Panicf("learner :%d receive an unexpected proposal mes: %+v", l.id, m)
		}
		//f.2 从accepted消息中来进行比对,如果接收的提案N > learner存入的提案N,需要重新学习;否则就忽略
		l.receiveAccept(m)
		//g. 如果半数以上的learner选择了提案N,则说明选择完毕
		accept, ok := l.chosen()
		if !ok {
			continue
		}
		log.Printf("learner :%d has chosen proposal : %v ", l.id, accept)
		l.value <- accept.proposalValue()
		return
	}
}

//g. 如果半数的learner选择了提案N,则说明选择完毕
func (l *learner) chosen() (accept, bool) {

	counts := make(map[int]int)
	accepts := make(map[int]accept)

	for _, accepted := range l.acceptors {
		// 统计learner接收提案的次数;为0说明没有接收过提案
		if accepted.proposalNumber() != 0 {
			counts[accepted.proposalNumber()]++
			accepts[accepted.proposalNumber()] = accepted
		}
	}

	for n, count := range counts {
		// quorum达到即返回
		if count >= l.quorum() {
			return accepts[n], true
		}
	}

	return mes{}, false
}

func (l *learner) quorum() int {
	return len(l.acceptors)/2 + 1
}

//f.2 从accepted消息中来进行比对,如果接收的提案N > learner存入的提案N,需要重新学习;否则就忽略
func (l *learner) receiveAccept(accepted mes) {
	a := l.acceptors[accepted.from]
	// 提案N < 接收的 N; 需要接收大于N的提案
	if a.proposalNumber() < accepted.n {
		log.Printf("learner %d has learned a new proposal mes: %+v", l.id, accepted)
		l.acceptors[accepted.from] = accepted
	}
}
```

看着大佬的代码理解了一遍;测试函数如下

```go
/*
@Time : 2019/8/27 22:17
@Author : Administrator
@File : test_paxos
@Software: GoLand
*/

package paxos

import (
	"fmt"
	"testing"
	"time"
)

type node struct {
	key  int
	next *node
}

func TestN(t *testing.T) {
	first := &node{}
	cur := &node{}
	lastSeq := 1
	id := 0
	// 保证生成的数不相同
	// 生成编号 N 的提案  测试了1亿级别的数据,耗时15.53s
	// --- PASS: TestN (15.53s)
	const num = 10000000

	for id = 0; id <= num; id++ {
		n := &node{key: lastSeq<<16 | id}
		if id == 0 {
			first = n
			cur = n
		} else {
			cur.next = n
			if cur.key == cur.next.key {
				t.Errorf("有重复元素: %+v", cur.key)
			}
			cur = n
			lastSeq++
		}
	}
	t.Log(first)
}

func TestSingleProposer(t *testing.T) {
	// 1,2,3,4,5 acceptors
	// 1001 proposer
	// 2001 learner
	pn := newPaxNet(1, 2, 3, 4, 5, 1001, 2001)
	ac := make([]*acceptor, 0)
	for i := 1; i <= 5; i++ {
		ac = append(ac, newAcceptor(i, pn.agentNet(i), 2001))
	}

	for _, a := range ac {
		go a.run()
	}
	wantValue := "hello world"
	p := newPropose(1001, wantValue, pn.agentNet(1001), 1, 2, 3, 4, 5)
	go p.run()

	l := newLearner(2001, pn.agentNet(2001), 1, 2, 3, 4, 5)
	go l.learn()
	v := l.GetValue()
	if v != wantValue {
		t.Errorf("value = %s wanted value = %s", v, wantValue)
	}
}

func TestTwoPropose(t *testing.T) {
	// 1,2,3 acceptors
	// 1001,1002 proposer
	// 2001 learner
	pn := newPaxNet(1, 2, 3, 1001, 1002, 2001)
	ac := make([]*acceptor, 0)
	for i := 1; i <= 3; i++ {
		ac = append(ac, newAcceptor(i, pn.agentNet(i), 2001))
	}
	for _, a := range ac {
		go a.run()
	}

	wantV1 := "hello world"
	p1 := newPropose(1001, wantV1, pn.agentNet(1001), 1, 2, 3)
	go p1.run()

	wantV2 := "hello world v2"
	// 提出提案N 此时lastSeq++;
	p2 := newPropose(1002, wantV2, pn.agentNet(1002), 1, 2, 3)
	go p2.run()

	l := newLearner(2001, pn.agentNet(2001), 1, 2, 3)
	go l.learn()
	va := l.GetValue()
	if va != wantV2 {
		t.Errorf("value = %s,wantValue = %s", va, wantV2)
	}

}

func TestNPropose(t *testing.T) {
	pn := newPaxNet(1, 2, 3, 1001, 1002, 1003, 2001, 2002)
	ac := make([]*acceptor, 0)
	for i := 1; i <= 3; i++ {
		ac = append(ac, newAcceptor(i, pn.agentNet(i), 2001, 2002))
	}
	for _, a := range ac {
		go a.run()
	}
	pp := make([]*proposer, 0)
	for i := 1001; i <= 1003; i++ {
		wantStr := "hello world v" + fmt.Sprint(i)
		pp = append(pp, newPropose(i, wantStr, pn.agentNet(i), 1, 2, 3))
	}

	for _, p := range pp {
		go p.run()
	}
	//这里模拟两个learner
	ln := make([]*learner, 0)
	for i := 2001; i <= 2002; i++ {
		ln = append(ln, newLearner(i, pn.agentNet(i), 1, 2, 3))
	}
	var v [2]string
	for k, l := range ln {
		go l.learn()
		v[k] = l.GetValue()
	}
	if v[0] != v[1] {
		t.Errorf("value = %s,wantValue = %s", v[0], v[1])
	}
	time.Sleep(500 * time.Millisecond)
}

```

### 验证结果:

```
=== RUN   TestNPropose
2019/08/28 18:58:29 nt send message :{from:1001 to:1 typ:1 n:66537 pren:0 value:}
2019/08/28 18:58:29 nt send message :{from:1001 to:2 typ:1 n:66537 pren:0 value:}
2019/08/28 18:58:29 nt recv message :{from:1001 to:1 typ:1 n:66537 pren:0 value:}
2019/08/28 18:58:29 acceptor 1 [promised: {from:0 to:0 typ:0 n:0 pren:0 value:}]  promised {from:1001 to:1 typ:1 n:66537 pren:0 value:}
2019/08/28 18:58:29 nt send message :{from:1 to:1001 typ:3 n:66537 pren:0 value:}
2019/08/28 18:58:29 nt recv message :{from:1001 to:2 typ:1 n:66537 pren:0 value:}
2019/08/28 18:58:29 acceptor 2 [promised: {from:0 to:0 typ:0 n:0 pren:0 value:}]  promised {from:1001 to:2 typ:1 n:66537 pren:0 value:}
2019/08/28 18:58:29 nt send message :{from:2 to:1001 typ:3 n:66537 pren:0 value:}
2019/08/28 18:58:29 nt send message :{from:1002 to:3 typ:1 n:66538 pren:0 value:}
2019/08/28 18:58:29 nt send message :{from:1002 to:1 typ:1 n:66538 pren:0 value:}
2019/08/28 18:58:29 nt recv message :{from:1002 to:1 typ:1 n:66538 pren:0 value:}
2019/08/28 18:58:29 acceptor 1 [promised: {from:1001 to:1 typ:1 n:66537 pren:0 value:}]  promised {from:1002 to:1 typ:1 n:66538 pren:0 value:}
2019/08/28 18:58:29 nt send message :{from:1 to:1002 typ:3 n:66538 pren:0 value:}
2019/08/28 18:58:29 nt recv message :{from:1 to:1002 typ:3 n:66538 pren:0 value:}
2019/08/28 18:58:29 proposer: 1002 received a new promise {from:1 to:1002 typ:3 n:66538 pren:0 value:}
2019/08/28 18:58:29 nt send message :{from:1003 to:1 typ:1 n:66539 pren:0 value:}
2019/08/28 18:58:29 nt send message :{from:1003 to:2 typ:1 n:66539 pren:0 value:}
2019/08/28 18:58:29 nt recv message :{from:1003 to:2 typ:1 n:66539 pren:0 value:}
2019/08/28 18:58:29 acceptor 2 [promised: {from:1001 to:2 typ:1 n:66537 pren:0 value:}]  promised {from:1003 to:2 typ:1 n:66539 pren:0 value:}
2019/08/28 18:58:29 nt send message :{from:2 to:1003 typ:3 n:66539 pren:0 value:}
2019/08/28 18:58:29 nt recv message :{from:2 to:1003 typ:3 n:66539 pren:0 value:}
2019/08/28 18:58:29 proposer: 1003 received a new promise {from:2 to:1003 typ:3 n:66539 pren:0 value:}
2019/08/28 18:58:29 nt recv message :{from:1002 to:3 typ:1 n:66538 pren:0 value:}
2019/08/28 18:58:29 acceptor 3 [promised: {from:0 to:0 typ:0 n:0 pren:0 value:}]  promised {from:1002 to:3 typ:1 n:66538 pren:0 value:}
2019/08/28 18:58:29 nt send message :{from:3 to:1002 typ:3 n:66538 pren:0 value:}
2019/08/28 18:58:29 nt recv message :{from:3 to:1002 typ:3 n:66538 pren:0 value:}
2019/08/28 18:58:29 proposer: 1002 received a new promise {from:3 to:1002 typ:3 n:66538 pren:0 value:}
2019/08/28 18:58:29 1002 promise 66538 reached quorum 2
proposer 1002: 2019/08/28 18:58:29 nt send message :{from:1002 to:1 typ:2 n:66538 pren:0 value:hello world v1002}
proposer 1002: 2019/08/28 18:58:29 nt send message :{from:1002 to:3 typ:2 n:66538 pren:0 value:hello world v1002}
2019/08/28 18:58:29 nt recv message :{from:1002 to:3 typ:2 n:66538 pren:0 value:hello world v1002}
2019/08/28 18:58:29 acceptor 3 [promised: {from:1002 to:3 typ:1 n:66538 pren:0 value:}, accept: {from:0 to:0 typ:0 n:0 pren:0 value:}]  accepted propose: {from:1002 to:3 typ:2 n:66538 pren:0 value:hello world v1002}
2019/08/28 18:58:29 nt send message :{from:3 to:2001 typ:4 n:66538 pren:0 value:hello world v1002}
2019/08/28 18:58:29 nt send message :{from:3 to:2002 typ:4 n:66538 pren:0 value:hello world v1002}
2019/08/28 18:58:29 nt recv message :{from:3 to:2001 typ:4 n:66538 pren:0 value:hello world v1002}
2019/08/28 18:58:29 learner 2001 has learned a new proposal mes: {from:3 to:2001 typ:4 n:66538 pren:0 value:hello world v1002}
2019/08/28 18:58:29 nt recv message :{from:1003 to:1 typ:1 n:66539 pren:0 value:}
2019/08/28 18:58:29 acceptor 1 [promised: {from:1002 to:1 typ:1 n:66538 pren:0 value:}]  promised {from:1003 to:1 typ:1 n:66539 pren:0 value:}
2019/08/28 18:58:29 nt send message :{from:1 to:1003 typ:3 n:66539 pren:0 value:}
2019/08/28 18:58:29 nt recv message :{from:1002 to:1 typ:2 n:66538 pren:0 value:hello world v1002}
2019/08/28 18:58:29 acceptor 1 [promised: {from:1003 to:1 typ:1 n:66539 pren:0 value:}] ignored propose mes: {from:1002 to:1 typ:2 n:66538 pren:0 value:hello world v1002}
2019/08/28 18:58:29 nt recv message :{from:1 to:1003 typ:3 n:66539 pren:0 value:}
2019/08/28 18:58:29 proposer: 1003 received a new promise {from:1 to:1003 typ:3 n:66539 pren:0 value:}
2019/08/28 18:58:29 1003 promise 66539 reached quorum 2
proposer 1003: 2019/08/28 18:58:29 nt send message :{from:1003 to:1 typ:2 n:66539 pren:0 value:hello world v1003}
proposer 1003: 2019/08/28 18:58:29 nt send message :{from:1003 to:2 typ:2 n:66539 pren:0 value:hello world v1003}
2019/08/28 18:58:29 nt recv message :{from:1003 to:2 typ:2 n:66539 pren:0 value:hello world v1003}
2019/08/28 18:58:29 acceptor 2 [promised: {from:1003 to:2 typ:1 n:66539 pren:0 value:}, accept: {from:0 to:0 typ:0 n:0 pren:0 value:}]  accepted propose: {from:1003 to:2 typ:2 n:66539 pren:0 value:hello world v1003}
2019/08/28 18:58:29 nt send message :{from:2 to:2001 typ:4 n:66539 pren:0 value:hello world v1003}
2019/08/28 18:58:29 nt send message :{from:2 to:2002 typ:4 n:66539 pren:0 value:hello world v1003}
2019/08/28 18:58:29 nt recv message :{from:2 to:2001 typ:4 n:66539 pren:0 value:hello world v1003}
2019/08/28 18:58:29 learner 2001 has learned a new proposal mes: {from:2 to:2001 typ:4 n:66539 pren:0 value:hello world v1003}
2019/08/28 18:58:29 nt recv message :{from:1003 to:1 typ:2 n:66539 pren:0 value:hello world v1003}
2019/08/28 18:58:29 acceptor 1 [promised: {from:1003 to:1 typ:1 n:66539 pren:0 value:}, accept: {from:0 to:0 typ:0 n:0 pren:0 value:}]  accepted propose: {from:1003 to:1 typ:2 n:66539 pren:0 value:hello world v1003}
2019/08/28 18:58:29 nt send message :{from:1 to:2001 typ:4 n:66539 pren:0 value:hello world v1003}
2019/08/28 18:58:29 nt send message :{from:1 to:2002 typ:4 n:66539 pren:0 value:hello world v1003}
2019/08/28 18:58:29 nt recv message :{from:1 to:2001 typ:4 n:66539 pren:0 value:hello world v1003}
2019/08/28 18:58:29 learner 2001 has learned a new proposal mes: {from:1 to:2001 typ:4 n:66539 pren:0 value:hello world v1003}
2019/08/28 18:58:29 learner :2001 has chosen proposal : {2 2001 4 66539 0 hello world v1003} 
2019/08/28 18:58:29 nt recv message :{from:3 to:2002 typ:4 n:66538 pren:0 value:hello world v1002}
2019/08/28 18:58:29 learner 2002 has learned a new proposal mes: {from:3 to:2002 typ:4 n:66538 pren:0 value:hello world v1002}
2019/08/28 18:58:29 nt recv message :{from:2 to:2002 typ:4 n:66539 pren:0 value:hello world v1003}
2019/08/28 18:58:29 learner 2002 has learned a new proposal mes: {from:2 to:2002 typ:4 n:66539 pren:0 value:hello world v1003}
2019/08/28 18:58:29 nt recv message :{from:1 to:2002 typ:4 n:66539 pren:0 value:hello world v1003}
2019/08/28 18:58:29 learner 2002 has learned a new proposal mes: {from:1 to:2002 typ:4 n:66539 pren:0 value:hello world v1003}
2019/08/28 18:58:29 learner :2002 has chosen proposal : {2 2002 4 66539 0 hello world v1003} 
2019/08/28 18:58:29 nt recv message :{from:1 to:1001 typ:3 n:66537 pren:0 value:}
2019/08/28 18:58:29 proposer: 1001 received a new promise {from:1 to:1001 typ:3 n:66537 pren:0 value:}
2019/08/28 18:58:29 nt recv message :{from:2 to:1001 typ:3 n:66537 pren:0 value:}
2019/08/28 18:58:29 proposer: 1001 received a new promise {from:2 to:1001 typ:3 n:66537 pren:0 value:}
2019/08/28 18:58:29 1001 promise 66537 reached quorum 2
proposer 1001: 2019/08/28 18:58:29 nt send message :{from:1001 to:1 typ:2 n:66537 pren:0 value:hello world v1001}
proposer 1001: 2019/08/28 18:58:29 nt send message :{from:1001 to:2 typ:2 n:66537 pren:0 value:hello world v1001}
2019/08/28 18:58:29 nt recv message :{from:1001 to:2 typ:2 n:66537 pren:0 value:hello world v1001}
2019/08/28 18:58:29 acceptor 2 [promised: {from:1003 to:2 typ:1 n:66539 pren:0 value:}] ignored propose mes: {from:1001 to:2 typ:2 n:66537 pren:0 value:hello world v1001}
2019/08/28 18:58:29 nt recv message :{from:1001 to:1 typ:2 n:66537 pren:0 value:hello world v1001}
2019/08/28 18:58:29 acceptor 1 [promised: {from:1003 to:1 typ:1 n:66539 pren:0 value:}] ignored propose mes: {from:1001 to:1 typ:2 n:66537 pren:0 value:hello world v1001}
--- PASS: TestNPropose (0.65s)
PASS
```

当有3个acceptor(1,2,3),2个proposer(1001,1002),2个learner(2001,2002),能证明learner最终能chosen统一的提案N.

### 参考

- 大佬的[github传送车](https://github.com/xiang90/paxos)
- [维基百科](https://zh.wikipedia.org/wiki/Paxos算法)

