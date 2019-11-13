---
title: "consul服务注册"
date: 2019-11-13T13:54:18+08:00
tags: [consul,cluster,go]
categories: [server]
---

 `$ 表示shell , # 表示注释, > 表示mysql` 

## consul 部署使用

使用consul，其主要有四大特性：

服务发现：利用服务注册，服务发现功能来实现服务治理。

健康检查：利用consul注册的检查检查函数或脚本来判断服务是否健康，若服务不存在则从注册中心移除该服务，减少故障服务请求。

k/v数据存储：存储kv数据，可以作为服务配置中心来使用。

多数据中心：可以建立多个consul集群通过inter网络进行互联，进一步保证数据可用性。

本人也是刚开始学习consul，感觉使用consul主要也就是两大作用，服务注册发现，配置共享。

### 使用

consul启动

```bash
$ nohup consul  agent -server -data-dir=/usr/local/consul-data/  -node=agent-one -bind=0.0.0.0 -bootstrap-expect=1 -client=0.0.0.0 -ui > /usr/local/consul-data/logs/consul.log 2>&1 &

-server 表示是以服务端身份启动
-bind 表示绑定到哪个ip（有些服务器会绑定多块网卡，可以通过bind参数强制指定绑定的ip）
-client 指定客户端访问的ip(consul有丰富的api接口，这里的客户端指浏览器或调用方)，0.0.0.0表示不限客户端ip
-bootstrap-expect=3 表示server集群最低节点数为3，低于这个值将工作不正常(注：类似zookeeper一样，通常集群数为奇数，方便选举，consul采用的是raft算法)
-data-dir 表示指定数据的存放目录（该目录必须存在）
-node 表示节点在web ui中显示的名称
```

查看当前的members

```bash
$ consul members
Node       Address              Status  Type    Build  Protocol  DC   Segment
agent-one  172.16.111.130:8301  alive   server  1.5.2  2         dc1  <all>
```

查看当前的leader

```
$ curl http://127.0.0.1:8500/v1/status/leader
"172.16.111.130:8300"
$ curl localhost:8500/v1/agent/services | python -m json.tool
{
    "hudongapps": {
        "Address": "",
        "EnableTagOverride": false,
        "ID": "hudongapps",
        "Meta": {},
        "Port": 22961,
        "Service": "hudongapps",
        "Tags": [
            "node"
        ],
        "Weights": {
            "Passing": 1,
            "Warning": 1
        }
    },
}
```

发现`service-msite`服务

```
$ curl http://127.0.0.1:8500/v1/catalog/service/service-msite | python -m json.tool
[
    {
        "Address": "172.16.111.130",
        "CreateIndex": 424242,
        "Datacenter": "dc1",
        "ID": "7f9c6b5d-8a32-c6f8-e091-0280f2efeddb",
        "ModifyIndex": 424242,
        "Node": "agent-one",
        "NodeMeta": {
            "consul-network-segment": ""
        },
        "ServiceAddress": "",
        "ServiceConnect": {},
        "ServiceEnableTagOverride": true,
        "ServiceID": "service-msite-22902",
        "ServiceKind": "",
        "ServiceMeta": {},
        "ServiceName": "service-msite",
        "ServicePort": 22902,
        "ServiceProxy": {},
        "ServiceProxyDestination": "",
        "ServiceTags": [
            "go"
        ],
        "ServiceWeights": {
            "Passing": 1,
            "Warning": 1
        },
        "TaggedAddresses": {
            "lan": "172.16.111.130",
            "wan": "172.16.111.130"
        }
    }
]

```

## golang 代码

注册逻辑

```go
package server

import (
	"log"

	"service-apis/config"
	pb "service-apis/proto-apis"

	. "github.com/youpenglai/mfwgo/registry"
	. "github.com/youpenglai/mfwgo/server"
	"google.golang.org/grpc"
)

func Setup() {
    // 获取配置, 存放name,port等相关信息
	cfg := config.GetConfig()

	// register service
	err := RegisterService(ServiceRegistration{ServiceName: cfg.ServiceName, Port: cfg.GRPCPort},
		ServiceRegisterType{CheckHealth: CheckHealth{Type: "grpc"}})
	if err != nil {
		log.Fatalf("register service error: %v", err)
		return
	}

	// start server
	grpcServer := NewGRPCServer(GRPCServerOption{Port: cfg.GRPCPort})
	addServer(grpcServer.GetServer())
	log.Fatalln(grpcServer.ListenAndServe())
}

func addServer(gServer *grpc.Server) {
	pb.RegisterApisServer(gServer, &ApisServer{})
}
```

启动服务

```go
package main

import (
	"service-apis/config"
	"service-apis/routers"
	"service-apis/server"
)

func main() {
	config.Setup()
	go server.Setup()  //注册服务
	routers.Setup()	   //起服务
}
```

其中注册和discover发现逻辑.

```
package registry

import (
	"encoding/json"
	"errors"
	"fmt"
	"os"
	"strconv"
)

var (
	consulIp   = "127.0.0.1"
	consulPort = 8500
)

// 支持从环境变量中获取
func initConsul() {
	ip := os.Getenv("MFW_CONSUL_IP")
	if ip != "" {
		consulIp = ip
	}
	port := os.Getenv("MFW_CONSUL_PORT")
	if port != "" {
		v, e := strconv.ParseInt(port, 10, 32)
		if e != nil {
			return
		}
		consulPort = int(v)
	}
}

const (
	deregisterInterval = "10m"
	tagGo              = "go"
	checkInterval      = "30s"
)

type ServiceRegisterInfo struct {
	ID                string      `json:"ID"`
	Name              string      `json:"Name"`
	Port              int64       `json:"Port"`
	Tags              []string    `json:"Tags"`
	Check             interface{} `json:"Check"`
	EnableTagOverride bool        `json:"EnableTagOverride"`
}

type HTTPCheck struct {
	DeregisterCriticalServiceAfter string `json:"DeregisterCriticalServiceAfter"`
	HTTP                           string `json:"HTTP"`
	Interval                       string `json:"Interval"`
}

type GRPCCheck struct {
	DeregisterCriticalServiceAfter string `json:"DeregisterCriticalServiceAfter"`
	GRPC                           string `json:"GRPC"`
	Interval                       string `json:"Interval"`
}

type ServiceHealthInfo struct {
	Node    Node              `json:"Node"`
	Service ConsulServiceInfo `json:"Service"`
	Checks  []Check           `json:"Checks"`
}

type Node struct {
	Address string `json:"Address"`
}

type Check struct {
	ServiceID   string `json:"ServiceID"`
	ServiceName string `json:"ServiceName"`
	Status      string `json:"Status"`
}

type ConsulServiceInfo struct {
	ID   string   `json:"ID"`
	Name string   `json:"Service"`
	Port int64    `json:"Port"`
	Tags []string `json:"Tags"`
}

type ConsulService struct{}

func NewConsulService() *ConsulService {
	return &ConsulService{}
}

func (c *ConsulService) Register(name string, port int64, healthType string) error {
	url := fmt.Sprintf("http://%s:%d/v1/agent/service/register", consulIp, consulPort)
	headers := map[string]string{
		"Content-Type": "application/json",
	}

	var check interface{}
	switch healthType {
	case "http":
		check = HTTPCheck{
			DeregisterCriticalServiceAfter: deregisterInterval,
			HTTP:                           fmt.Sprintf("http://%s:%d/health", consulIp, port),
			Interval:                       checkInterval,
		}
	case "grpc":
		check = GRPCCheck{
			DeregisterCriticalServiceAfter: deregisterInterval,
			GRPC:                           fmt.Sprintf("%s:%d", consulIp, port),
			Interval:                       checkInterval,
		}
	}

	register := ServiceRegisterInfo{
		Name:              name,
		Port:              port,
		ID:                fmt.Sprintf("%v-%v", name, port),
		Tags:              []string{tagGo},
		Check:             check,
		EnableTagOverride: true,
	}
	data, _ := json.Marshal(register)

	errMsg, err := HttpPutWithHeader(url, headers, data)
	if err != nil {
		return err
	}

	if string(errMsg) != "" {
		return errors.New(string(errMsg))
	}

	return c.GetServicesToCache(name)
}

func (c *ConsulService) GetServices(serviceName string) ([]*ServiceInfo, error) {
	url := fmt.Sprintf("http://%s:%d/v1/health/service/%s?passing", consulIp, consulPort, serviceName)
	headers := map[string]string{}

	result, err := HttpGetWithHeader(url, headers)
	if err != nil {
		return nil, err
	}

	var infos []ServiceHealthInfo
	if err := json.Unmarshal(result, &infos); err != nil {
		return nil, err
	}

	//if len(infos) == 0 {
	//	return nil, errors.New("not found service: " + serviceName)
	//}

	var validInfos []*ServiceInfo
	for _, info := range infos {
		validInfos = append(validInfos, &ServiceInfo{
			Address: info.Node.Address,
			Port:    info.Service.Port,
			Tags:    info.Service.Tags,
		})
	}

	return validInfos, nil
}

func (c *ConsulService) GetServicesToCache(serviceName string) error {
	infos, err := c.GetServices(serviceName)
	if err != nil {
		return err
	}
	// consul
	return gConsulCache.Set(serviceName, infos)
}

func init() {
	initConsul()
}
```

注册consul后, 查询具体使用curl来进行查询.

```bash
$ curl localhost:8500/v1/agent/services | python -m json.tool 
  "service-apis-22906": {
        "Address": "",
        "EnableTagOverride": true,
        "ID": "service-apis-22906",
        "Meta": {},
        "Port": 22906,
        "Service": "service-apis",
        "Tags": [
            "go"
        ],
        "Weights": {
            "Passing": 1,
            "Warning": 1
        }
    },
```

使用官方的api操作

```
package main
 
import (
    "github.com/gin-gonic/gin"
 
    consulapi "github.com/hashicorp/consul/api"
    "net/http"
    "fmt"
    "log"
)
 
func main() {
    r := gin.Default()
 
    // consul健康检查回调函数
    r.GET("/", func(c *gin.Context) {
        c.JSON(200, gin.H{
            "message": "ok",
        })
    })
 
 
    // 注册服务到consul
    ConsulRegister()
 
    // 从consul中发现服务
    ConsulFindServer()
 
    // 取消consul注册的服务
    //ConsulDeRegister()
 
 
    http.ListenAndServe(":8081", r)
}
 
// 注册服务到consul
func ConsulRegister()  {
    // 创建连接consul服务配置
    config := consulapi.DefaultConfig()
    config.Address = "127.0.0.1:8500"
    client, err := consulapi.NewClient(config)
    if err != nil {
        log.Fatal("consul client error : ", err)
    }
 
    // 创建注册到consul的服务到
    registration := new(consulapi.AgentServiceRegistration)
    registration.ID = "111"
    registration.Name = "go-consul-test"
    registration.Port = 8081
    registration.Tags = []string{"go-consul-test"}
    registration.Address = "10.13.153.128"
 
    // 增加consul健康检查回调函数
    check := new(consulapi.AgentServiceCheck)
    check.HTTP = fmt.Sprintf("http://%s:%d", registration.Address, registration.Port)
    check.Timeout = "5s"
    check.Interval = "5s"
    check.DeregisterCriticalServiceAfter = "30s" // 故障检查失败30s后 consul自动将注册服务删除
    registration.Check = check
 
    // 注册服务到consul
    err = client.Agent().ServiceRegister(registration)
}
 
// 取消consul注册的服务
func ConsulDeRegister()  {
    // 创建连接consul服务配置
    config := consulapi.DefaultConfig()
    config.Address = "127.0.0.1:8500"
    client, err := consulapi.NewClient(config)
    if err != nil {
        log.Fatal("consul client error : ", err)
    }
 
    client.Agent().ServiceDeregister("111")
}
 
// 从consul中发现服务
func ConsulFindServer()  {
    // 创建连接consul服务配置
    config := consulapi.DefaultConfig()
    config.Address = "127.0.0.1:8500"
    client, err := consulapi.NewClient(config)
    if err != nil {
        log.Fatal("consul client error : ", err)
    }
 
    // 获取所有service
    services, _ := client.Agent().Services()
    for _, value := range services{
        fmt.Println(value.Address)
        fmt.Println(value.Port)
    }
 
    fmt.Println("=================================")
    // 获取指定service
    service, _, err := client.Agent().Service("111", nil)
    if err == nil{
        fmt.Println(service.Address)
        fmt.Println(service.Port)
    }
 
}
 
func ConsulCheckHeath()  {
    // 创建连接consul服务配置
    config := consulapi.DefaultConfig()
    config.Address = "127.0.0.1:8500"
    client, err := consulapi.NewClient(config)
    if err != nil {
        log.Fatal("consul client error : ", err)
    }
 
    // 健康检查
    a, b, _ := client.Agent().AgentHealthServiceByID("111")
    fmt.Println(a)
    fmt.Println(b)
}
 
func ConsulKVTest()  {
    // 创建连接consul服务配置
    config := consulapi.DefaultConfig()
    config.Address = "127.0.0.1:8500"
    client, err := consulapi.NewClient(config)
    if err != nil {
        log.Fatal("consul client error : ", err)
    }
 
    // KV, put值
    values := "test"
    key := "go-consul-test/127.0.0.1:8100"
    client.KV().Put(&consulapi.KVPair{Key:key, Flags:0, Value: []byte(values)}, nil)
 
    // KV get值
    data, _, _ := client.KV().Get(key, nil)
    fmt.Println(string(data.Value))
 
    // KV list
    datas, _ , _:= client.KV().List("go", nil)
    for _ , value := range datas{
        fmt.Println(value)
    }
    keys, _ , _ := client.KV().Keys("go", "", nil)
    fmt.Println(keys)
}
```

 根据此功能专门做一个服务管理的模块，客户端注册服务到服务模块，服务管理去提供其他模块的服务发现的功能，同时跟监控结合，当服务不可用时，或服务不存在时，通过监控通知相关人员，我们也可以使用页面跟我们服务管理结合，通过前台服务录入形式进行注册服务.

参考

- [consul官方网站]( https://github.com/hashicorp/consul)