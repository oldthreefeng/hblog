---
title: "go-filecoin"
date: "2019-03-04 20:11:12"
tags: [filecoin,go]
categories: [github]
---

[TOC]

## IPFS 与 FileCoin

> IPFS 是一个网络协议，对标 HTTP 协议，中文叫做星际文件系统。IPFS 本质上是一种点对点的分布式文件系统， 旨在连接所有有相同的文件系统的计算机设备。在某些方面， IPFS 类似于 web, 但 web 是中心化的，而 IPFS 是一个单一的 Bittorrent 群集， 用 git 仓库分布式存储。换句话说， IPFS 提供了高吞吐量的内容寻址块存储模型， 具有内容寻址的超链接。这形成了一个广义的Merkle DAG 数据结构，可以用这个数据结构构建版本文件系统，区块链，甚至是永久性网站。IPFS 结合了分布式哈希表， 带有激励机制的块交换和自我认证命名空间。IPFS 没有单故障点， 节点不需要相互信任。

> Filecoin 是一个去中心化存储网络，它让云存储变成一个算法市场。这个市场运行在有着本地协议令牌（也叫做 Filecoin）的区块链。区块链中的矿工可以通过为客户提供存储来获取 Filecoin；相反的，客户可以通过花费 Filecoin 来雇佣矿工来存储或分发数据。和比特币一样，Filecoin 的矿工们为了巨大的奖励而竞争式挖区块，但 Filecoin 的挖矿效率是与存储活跃度成比例的，这直接为客户提供了有用的服务（不像比特币的挖矿仅是为了维护区块链的共识）。这种方式给矿工们创造了强大的激励，激励他们尽可能多的聚集存储器并且把它们出租给客户们。Filecoin 协议将这些聚集的资源编织成世界上任何人都能依赖的自我修复的存储网络。该网络通过复制和分散内容实现鲁棒性，同时自动检测和修复副本失败。客户可以选择复制参数来防范不同的威胁模型。该协议的云存储网络还提供了安全性，因为内容是在客户端端对端加密的，而存储提供者不能访问到解密秘钥。
>
> **当 Filecoin 与 IPFS 走在一起，Filecoin 则是运行在 IPFS 上面的一个激励层**

## go-filecoin安装

系统要求：  Linux and MacOS systems ，windows暂不支持

go-filecoin是go&rustc语言编写，且一些源码是放在Google上的，需要翻墙~

### go二进制安装

```
]$ wget https://dl.google.com/go/go1.11.5.linux-amd64.tar.gz
]$ tar xf go1.11.5.linux-amd64.tar.gz -C /usr/local/
]$ vim /etc/profile
##在最后行加入下面信息##
export GOROOT="/usr/local/go"
##这个很关键，go编译的gx工具都生成在GOPATH里面，如果不在$PATH里面，会报错##
export GOPATH="/root/go"
export PATH=$PATH:$GOROOT/bin:$GOPATH/bin
##configure vps proxy ##
export https_proxy=http://127.0.0.1:8118/
export http_proxy=http://127.0.0.1:8118/
]$ . /etc/profile
```

### rustc脚本安装

```
]$  curl https://sh.rustup.rs -sSf | sh
]$  source $HOME/.cargo/env
```

### proxy翻墙代理

```
]$ yum install python-pip
]$ pip install  git+https://github.com/shadowsocks/shadowsocks.git@master
]$ pip install --upgrade pip
]$ vim /etc/shadowsocks.json
]$ sslocal -c /etc/shadowsocks.json -d start 
]$ yum install -y privoxy
]$ vim /etc/privoxy/config 
]$ echo  'forward-socks5   /     127.0.0.1:1080 . '  > /etc/privoxy/config
]$ service privoxy start
]$ export http_proxy=http://127.0.0.1:8118/
]$ export https_proxy=http://127.0.0.1:8118/
```

### go-filecoin安装

```
cd ${GOPATH}/src/github.com/filecoin-project/go-filecoin
FILECOIN_USE_PRECOMPILED_RUST_PROOFS=true go run ./build/*.go deps
```

中间有报错缺少GLIBC_2.18,需要安装编译

```
]$  strings /usr/lib64/libc.so.6 | grep ^GLIBC_
]$  cd 
]$  curl -O http://ftp.gnu.org/gnu/glibc/glibc-2.18.tar.gz
]$  tar zxf glibc-2.18.tar.gz
]$  cd glibc-2.18/
]$  mkdir build
]$  cd build/
]$  ../configure --prefix=/usr
]$  make -j2 && make install 
]$  strings /usr/lib64/libc.so.6 | grep ^GLIBC_
]$  cd ${GOPATH}/src/github.com/filecoin-project/go-filecoin
]$   ./proofs/bin/paramcache
]$  go run ./build/main.go build
]$  go run ./build/main.go install 
```

安装好go-filecoin,可以开始了[Getting Started](https://github.com/filecoin-project/go-filecoin/wiki/Getting-Started),也可以[Mining-Filecoin](https://github.com/filecoin-project/go-filecoin/wiki/Mining-Filecoin)

编译安装大概花了1个小时，中间陆陆续续踩的坑~

## 参考

- [官方github](https://github.com/filecoin-project/go-filecoin/README.md)

