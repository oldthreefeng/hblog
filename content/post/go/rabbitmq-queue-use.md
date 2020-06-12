---
title: "RabbitMq消息队列使用"
date: 2019-11-07T13:54:18+08:00
tags: [queue,rabbitmq,cluster,haproxy,go]
categories: [server]
---

 `$ 表示shell , # 表示注释, > 表示mysql` 

## 什么是消息队列

消息（Message）是指在应用间传送的数据。消息可以非常简单，比如只包含文本字符串，也可以更复杂，可能包含嵌入对象。

消息队列（Message Queue）是一种应用间的通信方式，消息发送后可以立即返回，由消息系统来确保消息的可靠传递。
消息发布者只管把消息发布到 MQ 中而不用管谁来取，消息使用者只管从 MQ 中取消息而不管是谁发布的。这样发布者和使用者都不用知道对方的存在。

### 为什么需要消息队列

> 以常见的订单系统为例，用户点击【下单】按钮之后的业务逻辑可能包括：扣减库存、生成相应单据、发红包、发短信通知。在业务发展初期这些逻辑可能放在一起同步执行，随着业务的发展订单量增长，需要提升系统服务的性能，这时可以将一些不需要立即生效的操作拆分出来异步执行，比如发放红包、发短信通知等。
> 这种场景下就可以用 MQ ，在下单的主流程（比如扣减库存、生成相应单据）完成之后发送一条消息到 MQ 让主流程快速完结，而由另外的单独线程拉取MQ的消息（或者由 MQ 推送消息），当发现 MQ 中有发红包或发短信之类的消息时，执行相应的业务逻辑。

以上用于业务解耦的情况，其它常见场景包括最终一致性、广播、错峰流控...

### RabbitMq 

> RabbitMQ这款消息队列中间件产品本身是基于Erlang编写，Erlang语言天生具备分布式特性（通过同步Erlang集群各节点的magic cookie来实现）。因此，RabbitMQ天然支持Clustering。

> AMQP ：Advanced Message Queue，高级消息队列协议。它是应用层协议的一个开放标准，为面向消息的中间件设计，基于此协议的客户端与消息中间件可传递消息，并不受产品、开发语言等条件的限制。

**消息模型**

所有 MQ 产品从模型抽象上来说都是一样的过程：

消费者（consumer）订阅某个队列。生产者（producer）创建消息，然后发布到队列（queue）中，最后将消息发送到监听的消费者。

- Message

消息，消息是不具名的，它由消息头和消息体组成。消息体是不透明的，而消息头则由一系列的可选属性组成，这些属性包括routing-key（路由键）、priority（相对于其他消息的优先权）、delivery-mode（指出该消息可能需要持久性存储）等。

- Publisher

消息的生产者，也是一个向交换器发布消息的客户端应用程序。

- Exchange

交换器，用来接收生产者发送的消息并将这些消息路由给服务器中的队列。

- Binding

绑定，用于消息队列和交换器之间的关联。一个绑定就是基于路由键将交换器和消息队列连接起来的路由规则，所以可以将交换器理解成一个由绑定构成的路由表。

- Queue

消息队列，用来保存消息直到发送给消费者。它是消息的容器，也是消息的终点。一个消息可投入一个或多个队列。消息一直在队列里面，等待消费者连接到这个队列将其取走。

- Connection

网络连接，比如一个TCP连接。

- Channel

信道，多路复用连接中的一条独立的双向数据流通道。信道是建立在真实的TCP连接内地虚拟连接，AMQP 命令都是通过信道发出去的，不管是发布消息、订阅队列还是接收消息，这些动作都是通过信道完成。因为对于操作系统来说建立和销毁 TCP 都是非常昂贵的开销，所以引入了信道的概念，以复用一条 TCP 连接。

- Consumer

消息的消费者，表示一个从消息队列中取得消息的客户端应用程序。

- Virtual Host

虚拟主机，表示一批交换器、消息队列和相关对象。虚拟主机是共享相同的身份认证和加密环境的独立服务器域。每个 vhost 本质上就是一个 mini 版的 RabbitMQ 服务器，拥有自己的队列、交换器、绑定和权限机制。vhost 是 AMQP 概念的基础，必须在连接时指定，RabbitMQ 默认的 vhost 是 / 。

- Broker

表示消息队列服务器实体。

Exchange类型

目前是有四种类型, 分别为direct, fanout, topic, header.

- direct

消息中的路由键（routing key）如果和 Binding 中的 binding key 一致， 交换器就将消息发到对应的队列中。
路由键与队列名完全匹配，如果一个队列绑定到交换机要求路由键为"test"，则只转发 routing key 标记为"dog"的消息，不会转发"test.puppy"，也不会转发"test.guard"等等。它是完全匹配、单播的模式。

### golang实例

项目结构

```bash
rabbitmq]# tree
.
├── go.mod
├── go.sum
├── main.go
├── receive
│   └── receiev.go
└── send
    └── send.go
```

`send.go` 作为消息的产生者, 采用的是routing key 和 queue名称相同.

```go
package send

import (
	"fmt"
	"github.com/streadway/amqp" //根据实际情况来定
	"log"
	"strconv"
	"time"
)

//创建一个返回错误打印日志的函数
func FailOnError(err error, msg string) {
	if err != nil {
		log.Fatalf("%s: %s", msg, err)
	}
}
func Send() {
	//打开一个连接
	conn, err := amqp.Dial("amqp://rabbit:123456@192.168.133.128:5672")
	FailOnError(err, "failed to connect to RabbitMQ")
	defer conn.Close()

	//打开一个通道
	ch, err := conn.Channel()
	FailOnError(err, "failed to open a channel")
	defer ch.Close()

	q, err := ch.QueueDeclare(
		"test rabbitMq number", // name
		false,                  // durable
		false,                  // delete when usused
		false,                  // exclusive
		false,                  // no-wait
		nil,                    // arguments
	)
	FailOnError(err, "Failed to declare a queue")
	fmt.Println("start send message !")
	for i := 0; i < 60; i++ {
		body := "hello, rabbitMq  " + strconv.Itoa(i)
		err = ch.Publish(
			"",
			q.Name, // routing key 和 queue name 相同,  路由键与队列名完全匹配,完成转发
			false,
			false,
			amqp.Publishing{
				ContentType: "text/plain",
				Body:        []byte(body),
			})
		FailOnError(err, "Failed to publish a message")
		time.Sleep(1 * time.Millisecond)
	}

}
```

`receive.go` 作为消息的消费者, 实时消费队列中的消息; 一直阻塞在等待消息队列

```go
package receive

import (
	"github.com/streadway/amqp"
	"gogs.wangke.co/go/rabbitmq/send"
	"log"
	"time"
)

//创建一个返回错误打印日志的函数

func Receive() {
	conn, err := amqp.Dial("amqp://rabbit:123456@192.168.133.128:5672")
	send.FailOnError(err, "Failed to connect to RabbitMQ")
	defer conn.Close()

	ch, err := conn.Channel()
	send.FailOnError(err, "Failed to open a channel")
	defer ch.Close()

	q, err := ch.QueueDeclare(
		"test rabbitMq number", // name
		false,                  // durable
		false,                  // delete when usused
		false,                  // exclusive
		false,                  // no-wait
		nil,                    // arguments
	)
	send.FailOnError(err, "Failed to declare a queue")
	msgs, err := ch.Consume(
		q.Name, // queue
		"",     // consumer
		true,   // auto-ack
		false,  // exclusive
		false,  // no-local
		false,  // no-wait
		nil,    // args
	)
	send.FailOnError(err, "Failed to register a consumer")

	forever := make(chan bool)
	for i := 0; i < 60; i++ {
		go func() {
			for d := range msgs {
				log.Printf("Received a message: %s", d.Body)
			}
		}()
		time.Sleep(30 * time.Millisecond)
	}
	log.Printf("Waiting for messages. To exit press CTRL+C")
	<-forever
}
```

主程序`main.go`, 先产生消息, 然后消费消息.

```go
package main

import (
	"gogs.wangke.co/go/rabbitmq/receive"
	"gogs.wangke.co/go/rabbitmq/send"
)

func main() {
	send.Send()
	receive.Receive()
}
```

modules

```mod
module gogs.wangke.co/go/rabbitmq

go 1.13

require github.com/streadway/amqp v0.0.0-20190827072141-edfb9018d271

```


编译并运行

```bash 
## 允许go modules
$ go env -w GO111MODULE=on
$ go env -w GOPROXY="https://goproxy.cn,direct"
## git clone 项目并运行.
$ git clone http://gogs.wangke.co/go/rabbitmq.git
$ cd rabbitmq &&  go run main.go
start send message !
2019/11/07 01:09:48 Received a message: hello, rabbitMq  0
2019/11/07 01:09:48 Received a message: hello, rabbitMq  1
2019/11/07 01:09:48 Received a message: hello, rabbitMq  2
2019/11/07 01:09:48 Received a message: hello, rabbitMq  3
2019/11/07 01:09:48 Received a message: hello, rabbitMq  4
2019/11/07 01:09:48 Received a message: hello, rabbitMq  5
2019/11/07 01:09:48 Received a message: hello, rabbitMq  6
2019/11/07 01:09:48 Received a message: hello, rabbitMq  7
2019/11/07 01:09:48 Received a message: hello, rabbitMq  8
2019/11/07 01:09:48 Received a message: hello, rabbitMq  9
2019/11/07 01:09:50  [*] Waiting for messages. To exit press CTRL+C

```

登陆本地的`rabbitmq web`客户端, 查看queue相关信息. 

![](https://pic.fenghong.tech/rabbitmq/rabbit-test-queue.png)
