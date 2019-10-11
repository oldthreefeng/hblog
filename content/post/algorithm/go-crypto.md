---
title: "go encrypt&decrypt"
date: 2019-08-28 18:10:10
tags: 
  - hash
  - rsa
  - ssh
  - crypto
categories: [algorithm] 
---

[TOC]

# 密码学简介与`Golang`的加密库`Crypto`的使用

> 据记载，公元前3200-1200年,古代埃及法老的墓上的象形文字
>
> 公元前500, 斯巴达人在军事上用于加密,发明了"天书"密码(Scytale).,发送者把一条羊皮螺旋形缠绕在圆柱木棒上,核心就是置换.
>
> 公元前400年，古希腊人发明了置换密码。
>
> 公元前50年,古罗马凯撒发明Caesar密码,代替密码
>
> 1881年世界上的第一个电话保密专利出现。
>
> 1883年Kerckhoffs第一次明确的提出了密码编码的原则: 加密算法应该建立在算法的公开不影响明文和秘钥的安全,即密码算法的安全性仅依赖于对秘钥的保密.
>
> 在第二次世界大战期间，德国军方启用“恩尼格玛”密码机，密码学在战争中起着非常重要的作用, 这段历史很有趣,建议看看[恩格玛机破解历史](https://www.zhihu.com/question/28397034),[Enigma加密视频](https://www.bilibili.com/video/av2883992),[Enigma解密视频](https://www.bilibili.com/video/av2884019)。
>
> 1949年,Shannon发表了"The Communication Theory of Secret Systems"这篇论文定义了理论安全性,提出了扩散和混淆原则,奠定了密码学的理论基础,密码学由艺术向科学科学的转变.
>
> 1976年Diffie & Hellman的 "New Directions in Cryptography",提出了公钥密码的概念.
>
> 随着信息化和数字化社会的发展，人们对信息安全和保密的重要性认识不断提高，于是在1997年，美国国家标准局公布实施了“美国数据加密标准（DES）”，民间力量开始全面介入密码学的研究和应用中，采用的加密算法有DES、RSA、SHA等。随着对加密强度需求的不断提高，近期又出现了AES、ECC等。
>
> 1994年,Shor提出了量子计算机模型下分解大整数和求离散对数的多项式时间算法.
>
> 2000年,AES正式取代了DES成为了新的加密标准;
>
> 后量子密码...

### 密码分析学

- 密码攻击

>1. 穷举攻击: 通过试遍所有的秘钥来破译
>2. 统计分析攻击: 通过分析秘闻和明文的统计规律来破译
>3. 解密变换攻击: 针对加密变换的数学基础,通过数学求解设法找到解密变换

- 无条件安全和计算上安全

>1. 无论截获多少密文,都没有足够信息来确定唯一明文,则该密码是无条件安全
>2. 使用有效资源对一个密码系统进行分析而未破译,则改密码是强的或者计算是安全

### 密码学的目的

- 保密性：防止用户的标识或数据被读取。
- 数据完整性：防止数据被更改。
- 身份验证：确保数据发自特定的一方。

### 加密算法

根据密钥类型不同将现代密码技术分为两类：

- 对称加密算法: 加密和解密均采用同一把秘密钥匙。
- 非对称加密算法: 有2把密钥,公钥和私钥, 公钥加密, 私钥解密。

### 非对称加密

公钥加密,秘钥是成对出现的,公钥公开给所有人,私钥自己留存,用公钥加密数据，只能使用与之配对的私钥解密；反之亦然;

- RSA: 由RSA公司发明，是一个支持变长密钥的公共密钥算法，需要加密的文件块的长度也是可变的；
- DSA(Digital Signature Algorithm): 数字签名算法，是一种标准的DSS(数字签名标准)；
- ECC(Elliptic Curves Cryptography): 椭圆曲线密码编码学。
- ECDSA(Elliptic Curve Digital Signature Algorithm): 基于椭圆曲线的DSA签名算法.

### 散列算法

散列是信息的提炼，通常其长度要比信息小得多，且为一个固定长度。加密性强的散列一定是不可逆的，这就意味着通过散列结果，无法推出任何部分的原始信息。任何输入信息的变化，哪怕仅一位，都将导致散列结果的明显变化，这称之为雪崩效应。散列还应该是防冲突的，即找不出具有相同散列结果的两条信息。具有这些特性的散列结果就可以用于验证信息是否被修改。常用于保证数据完整性
单向散列函数一般用于产生消息摘要，密钥加密等，常见的有：

- MD5(Message Digest Algorithm 5): 是RSA数据安全公司开发的一种单向散列算法。
- SHA(Secure Hash Algorithm): 可以对任意长度的数据运算生成一个160位的数值；

#### MD5

MD5即Message-Digest Algorithm 5（信息-摘要算法5），用于确保信息传输完整一致。是计算机广泛使用的杂凑算法之一（又译摘要算法、哈希算法），主流编程语言普遍已有MD5实现。将数据（如汉字）运算为另一固定长度值，是杂凑算法的基础原理，MD5的前身有MD2、MD3和MD4

#### SHA-1

在1993年，安全散列算法（SHA）由美国国家标准和技术协会(NIST)提出，并作为联邦信息处理标准（FIPS PUB 180）公布；1995年又发布了一个修订版FIPS PUB 180-1，通常称之为SHA-1。SHA-1是基于MD4算法的，并且它的设计在很大程度上是模仿MD4的。现在已成为公认的最安全的散列算法之一，并被广泛使用。
SHA-1是一种数据加密算法，该算法的思想是接收一段明文，然后以一种不可逆的方式将它转换成一段（通常更小）密文，也可以简单的理解为取一串输入码（称为预映射或信息），并把它们转化为长度较短、位数固定的输出序列即散列值（也称为信息摘要或信息认证代码）的过程。
该算法输入报文的最大长度不超过264位，产生的输出是一个160位的报文摘要。输入是按512 位的分组进行处理的。SHA-1是不可逆的、防冲突，并具有良好的雪崩效应。
sha1是SHA家族的五个算法之一(其它四个是SHA-224、SHA-256、SHA-384，和SHA-512)

#### HMac

Hmac算法也是一种哈希算法，它可以利用MD5或SHA1等哈希算法。不同的是，Hmac还需要一个密钥, 只要密钥发生了变化，那么同样的输入数据也会得到不同的签名，因此，可以把Hmac理解为用随机数“增强”的哈希算法。

```go
package main

import (
	"crypto/hmac"
	"crypto/sha1"
	"encoding/hex"
	"fmt"
	"io"
)

func main() {
	h := sha1.New()
	h.Write([]byte("hello world"))
	//sha1Hash散列
	fmt.Printf("%x\n", h.Sum(nil))
	//sha1Hmac散列
	fmt.Println(getHmacCode("hello world"))
}

func getHmacCode(s string) string {
	h := hmac.New(sha1.New, []byte(s))
	_, _ = io.WriteString(h, s)
	return hex.EncodeToString(h.Sum(nil))
}

```

#### SHA-1与MD5的比较

因为二者均由MD4导出，SHA-1和MD5彼此很相似。相应的，他们的强度和其他特性也是相似，但还有以下几点不同：

- 对强行供给的安全性：最显著和最重要的区别是SHA-1摘要比MD5摘要长32 位。使用强行技术，产生任何一个报文使其摘要等于给定报摘要的难度对MD5是2128数量级的操作，而对SHA-1则是2160数量级的操作。这样，SHA-1对强行攻击有更大的强度。
- 对密码分析的安全性：由于MD5的设计，易受密码分析的攻击，SHA-1显得不易受这样的攻击。
- 速度：在相同的硬件上，SHA-1的运行速度比MD5慢。

### 椭圆加密算法

#### DH

DH全称是:`Diffie-Hellman`, 是一种确保共享KEY安全穿越不安全网络的方法，它是OAKLEY的一个组成部分。Whitefield与Martin Hellman在1976年提出了一个奇妙的密钥交换协议，称为Diffie-Hellman密钥交换协议/算法(Diffie-Hellman Key Exchange/Agreement Algorithm).这个机制的巧妙在于需要安全通信的双方可以用这个方法确定对称密钥。然后可以用这个密钥进行加密和解密。
DH依赖于计算离散对数的难度, 大概过程如下:

> 可以如下定义离散对数：首先定义一个素数p的原根，为其各次幂产生从1 到p-1的所有整数根，也就是说，如果a是素数p的一个原根，那么数值 a mod p,a2 mod p,…,ap-1 mod p 是各不相同的整数，并且以某种排列方式组成了从1到p-1的所有整数. 对于一个整数b和素数p的一个原根a，可以找到惟一的指数i，使得 b = a^i mod p 其中0 ≤ i ≤ （p-1） 指数i称为b的以a为基数的模p的离散对数或者指数.该值被记为inda,p(b).

#### ECDH

全称是`Elliptic Curve Diffie-Hellman`, 是DH算法的加强版, 基于椭圆曲线难题加密, 现在是主流的密钥交换算法。
ECC是建立在基于椭圆曲线的离散对数的难度, 大概过程如下:

> 给定椭圆曲线上的一个点P，一个整数k，求解Q=kP很容易；给定一个点P、Q，知道Q=kP，求整数k确是一个难题。ECDH即建立在此数学难题之上

椭圆曲线算法因参数不同有多种类型, 这个网站列出了现阶段那些ECC是相对安全的:[椭圆曲线算法安全列表](http://safecurves.cr.yp.to/), 而curve25519便是其中的佼佼者。
Curve25519/Ed25519/X25519是著名密码学家Daniel J. Bernstein在2006年独立设计的椭圆曲线加密/签名/密钥交换算法, 和现有的任何椭圆曲线算法都完全独立。
特点是：

- 完全开放设计: 算法各参数的选择直截了当，非常明确，没有任何可疑之处，相比之下目前广泛使用的椭圆曲线是NIST系列标准，方程的系数是使用来历不明的随机种子 c49d3608 86e70493 6a6678e1 139d26b7 819f7e90 生成的，非常可疑，疑似后门；
- 高安全性： 一个椭圆曲线加密算法就算在数学上是安全的，在实用上也并不一定安全，有很大的概率通过缓存、时间、恶意输入摧毁安全性，而25519系列椭圆曲线经过特别设计，尽可能的将出错的概率降到了最低，可以说是实践上最安全的加密算法。例如，任何一个32位随机数都是一个合法的X25519公钥，因此通过恶意数值攻击是不可能的，算法在设计的时候刻意避免的某些分支操作，这样在编程的时候可以不使用if ，减少了不同if分支代码执行时间不同的时序攻击概率，相反， NIST系列椭圆曲线算法在实际应用中出错的可能性非常大，而且对于某些理论攻击的免疫能力不高， Bernstein 对市面上所有的加密算法使用12个标准进行了考察， 25519是几乎唯一满足这些标准的 <http://t.cn/RMGmi1g> ；
- 速度快: 25519系列曲线是目前最快的椭圆曲线加密算法，性能远远超过NIST系列，而且具有比P-256更高的安全性；
- 作者功底深厚: Daniel J. Bernstein是世界著名的密码学家，他在大学曾经开设过一门 UNIX 系统安全的课程给学生，结果一学期下来，发现了 UNIX 程序中的 91 个安全漏洞；他早年在美国依然禁止出口加密算法时，曾因为把自己设计的加密算法发布到网上遭到了美国政府的起诉，他本人抗争六年，最后美国政府撤销所有指控，目前另一个非常火的高性能安全流密码 ChaCha20 也是出自 Bernstein 之手；
- 下一代的标准: 25519系列曲线自2006年发表以来，除了学术界无人问津， 2013 年爱德华·斯诺登曝光棱镜计划后，该算法突然大火，大量软件，如OpenSSH都迅速增加了对25519系列的支持，如今25519已经是大势所趋，可疑的NIST曲线迟早要退出椭圆曲线的历史舞台，目前， RFC增加了SSL/TLS对X25519密钥交换协议的支持，而新版 OpenSSL 1.1也加入支持，是摆脱老大哥的第一步，下一步是将 Ed25519做为可选的TLS证书签名算法，彻底摆脱NIST
- 这里需要指出下golang的标准库的crypto里的椭圆曲线实现了这4种(elliptic文档): P224/P256/P384/P521, 而curve25519是单独实现的, 他不在标准库中: `golang.org/x/crypto/curve25519`

## go 加密解密之RSA

> 安全之道,对应通用的加密算法,很多语言都有实现. 对于RSA算法本身,请自行google.
>
> 在1976年，由于对称加密算法已经不能满足需要，Diffie 和Hellman发表了一篇叫《密码学新动向》的文章，介绍了公匙加密的概念，由Rivet、Shamir、Adelman提出了RSA算法。
RSA是目前最有影响力的公钥加密算法，它能够抵抗到目前为止已知的绝大多数密码攻击，已被ISO推荐为公钥数据加密标准。
>
> 本文讨论的Go RSA,均在win10操作系统上完成

### 概要

RSA是一个非对称加密算法,通过公钥加密,私钥解密.

```shell
$ openssl genrsa -out private.pem 1024
$ openssl rsa -in private.pem -pubout -out public.pem
```

这样,便生成了秘钥,当然很多工具都提高了RSA的秘钥生成方法,编程语言大部分也提供了API来生成.加密解密涉及很多标准,需要的时候可以去临时学一下.

### GO生成RSA

这里提供golang方法来生成,思路如下:

- `private.pem`生成

在`crypto/rsa`包中有一个函数：

>  `func GenerateKey(random io.Reader, bits int) (priv *PrivateKey, err error)`

> 官方解释 : GenerateKey generates an RSA keypair of the given bit size using the random source random (for example, crypto/rand.Reader).

该函数中，`random`可以直接传`crypto/rand`中的`rand.Reader`，而bits是密钥长度。



这样得到了一个`PrivateKey`类型的指针

```go
type PrivateKey struct {
	PublicKey            // public part.
	D         *big.Int   // private exponent
	Primes    []*big.Int // prime factors of N, has >= 2 elements.

	// Precomputed contains precomputed values that speed up private
	// operations, if available.
	Precomputed PrecomputedValues
}

// A PublicKey represents the public part of an RSA key.
type PublicKey struct {
	N *big.Int // modulus
	E int      // public exponent
}
```

得到`*PrivateKey`指针后,`MarshalPKCS1PrivateKey`生成一个ASN.1的加密form.

> `func MarshalPKCS1PrivateKey(key *rsa.PrivateKey) []byte `

> 官方解释: MarshalPKCS1PrivateKey converts a private key to ASN.1 DER encoded form.

将加密的ASN.1存入Block里面,`Block`是`pem`包里面的一个结构体,是标准的PEM格式,一个block代表的是PEM编码的结构，关于PEM，请查阅相关资料,有三个参数,一个是`Type`,是一个`string`类型,
`Bytes`即我们刚才得到的ANS.1的DER字节切片.其实这里的秘钥已经生成完毕.

```go
// A Block represents a PEM encoded structure.
//
// The encoded form is:
//    -----BEGIN Type-----
//    Headers
//    base64-encoded Bytes
//    -----END Type-----
// where Headers is a possibly empty sequence of Key: Value lines.
type Block struct {
	Type    string            // The type, taken from the preamble (i.e. "RSA PRIVATE KEY").
	Headers map[string]string // Optional headers.
	Bytes   []byte            // The decoded bytes of the contents. Typically a DER encoded ASN.1 structure.
}
```

将其写入文件,生成文件句柄,将block写入文件即可,这里利用`pem`包里面的Encode方法.私钥生成完毕.

```
// Encode writes the PEM encoding of b to out.
func Encode(out io.Writer, b *Block) error
```

- `public.pem`生成

在生成`*PrivateKey`秘钥的时候,这个结构体已经含有了`PublicKey`这个结构了,需要对`PublicKey`导出的格式进行定义,这里有两种导出格式: `PKCS#1, ASN.1 DER form` 和 `DER-encoded PKIX`,都是可以操作的.这里演示前者即ASN.1 DER

> `func MarshalPKCS1PublicKey(key *rsa.PublicKey) []byte`

> 官方解释: MarshalPKCS1PublicKey converts an RSA public key to PKCS#1, ASN.1 DER form.

> `func MarshalPKIXPublicKey(pub interface{}) ([]byte, error) `

> 官方解释: MarshalPKIXPublicKey serialises a public key to DER-encoded PKIX format.

生成好`PKCS#1, ASN.1 DER form`格式之后,将其存入block里面,不过这次的`Type`为`"PUBLIC KEY"`,之后,生成文件句柄,将block写入文件即可,这里同样利用`pem`包里面的Encode方法.私钥生成完毕.

完整代码如下:

```go
/*
@Time : 2019/8/31 22:40
@Author : louis
@File : rsaGenerate
@Software: GoLand
*/

package main

import (
	"crypto/rand"
	"crypto/rsa"
	"crypto/x509"
	"encoding/pem"
	"flag"
	"log"
	"os"
)


func generateKey(bits int, path string) (err error) {
	privateKey, err := rsa.GenerateKey(rand.Reader, bits)
	if err != nil {
		return err
	}
	// MarshalPKCS1PrivateKey converts a private key to ASN.1 DER encoded form.
	derStream := x509.MarshalPKCS1PrivateKey(privateKey)
	block := &pem.Block{
		Type:  "RSA PRIVATE KEY",
		Bytes: derStream,
	}
	// file,err := os.OpenFile(name,O_RDWR|O_CREATE|O_TRUNC, 0666)
	file, err := os.Create(path + "private.pem")
	if err != nil {
		return err
	}
	defer file.Close()
	err = pem.Encode(file, block)
	if err != nil {
		return err
	}

	publicKey := &privateKey.PublicKey
	// MarshalPKIXPublicKey serialises a public key to DER-encoded PKIX format.
	//derPub, err := x509.MarshalPKIXPublicKey(publicKey)
	// MarshalPKCS1PublicKey converts an RSA public key to PKCS#1, ASN.1 DER form.
	derPub := x509.MarshalPKCS1PublicKey(publicKey)
	block = &pem.Block{
		Type:  "PUBLIC KEY",
		Bytes: derPub,
	}
	pubFile, err := os.Create(path + "public.pem")
	if err != nil {
		return err
	}
	//pubFile1, err := os.Create("sshgo/public1.pem")
	//if err != nil {
	//	log.Fatalln(err)
	//}
	err = pem.Encode(pubFile, block)
	if err != nil {
		return err
	}
	//err = pem.Encode(pubFile1,block1)
	//if err != nil {
	//	log.Fatalln(err)
	//}
	return nil
}


func main() {
	var bits int
	var path string
	flag.IntVar(&bits, "b", 1024, "rsa key length")
	flag.StringVar(&path,"p","sshgo/","rsa key path")
	if err := generateKey(bits, path); err != nil {
		log.Fatalln("rsa generate failed")
	}
	log.Println("rsa key generate success")
}

```

### GO加密解密数据

如何利用`PublicKey`和`PrivateKey`来实现加密解密呢?

- 加密(利用pubKey加密)

在上面生成秘钥的时候用到了一个`pem.Encode`方法,相应的是不是有`pem.Decode`方法呢?还真必须有~~

```go
// Decode will find the next PEM formatted block (certificate, private key
// etc) in the input. It returns that block and the remainder of the input. If
// no PEM data is found, p is nil and the whole of the input is returned in
// rest.
func Decode(data []byte) (p *Block, rest []byte) {}
```
将秘钥文件读取,`ioutil.ReadFile`这个方法,返回`[]byte error `即可完美衔接`Decode`方法,得到`*block`指针.

得到指针又该如何使用呢?前面已经说过,block结构里面有`Bytes`字段,是一个form结构,比如`PKCS#1, ASN.1 DER form` 和 `DER-encoded PKIX`,依旧利用前者,如果你的PublicKey利用ASN.1生成的,就选择相应的方法进行分解. 这个方法在`crypto/x509`

> 官方解释: ParsePKCS1PublicKey parses a PKCS#1 public key in ASN.1 DER form.

> func ParsePKCS1PublicKey(der []byte) (*rsa.PublicKey, error) 

```go
// ParsePKIXPublicKey parses a DER encoded public key. These values are
// typically found in PEM blocks with "BEGIN PUBLIC KEY".
//
// Supported key types include RSA, DSA, and ECDSA. Unknown key
// types result in an error.
//
// On success, pub will be of type *rsa.PublicKey, *dsa.PublicKey,
// or *ecdsa.PublicKey.
func ParsePKIXPublicKey(derBytes []byte) (pub interface{}, err error) 
//如果采用这个方法,请记得用类型断言一下,比如
// pub.(*rsa.PublicKey)
```

获得`*rsa.PublicKey`后,利用`EncryptPKCS1v15`即可完成加密.返回一个[]byte,字节切片.解密过程原理一直

```go
// EncryptPKCS1v15 encrypts the given message with RSA and the padding
// scheme from PKCS#1 v1.5.  The message must be no longer than the
// length of the public modulus minus 11 bytes.
//
// The rand parameter is used as a source of entropy to ensure that
// encrypting the same message twice doesn't result in the same
// ciphertext.
//
// WARNING: use of this function to encrypt plaintexts other than
// session keys is dangerous. Use RSA OAEP in new protocols.
func EncryptPKCS1v15(rand io.Reader, pub *PublicKey, msg []byte) ([]byte, error) 
```

- 完整代码

```go
/*
@Time : 2019/8/31 11:58
@Author : louis
@File : rsaEnDecode
@Software: GoLand
*/

package main

import (
	"crypto/rand"
	"crypto/rsa"
	"crypto/x509"
	"encoding/base64"
	"encoding/pem"
	"fmt"
	"io/ioutil"
	"log"
)

//怎么将openssl生成的密钥文件解析到公钥和私钥实例呢？

//rsaEncrypt 加密数据
func rsaEncrypt(ori string) ([]byte, error) {
	//data, err := ioutil.ReadFile("sshgo/public.pem")
	data, err := ioutil.ReadFile("sshgo/public.pem")
	if err != nil {
		return nil, err
	}
	block, _ := pem.Decode(data)
	if block == nil {
		return nil, err
	}
	//pubIn, err := x509.ParsePKIXPublicKey(block.Bytes)
	pub, err := x509.ParsePKCS1PublicKey(block.Bytes)
	if err != nil {
		return nil, err
	}
	//pub := pubIn.(*rsa.PublicKey)
	return rsa.EncryptPKCS1v15(rand.Reader, pub, []byte(ori))

}

func rsaDecrypt(encrypt []byte) ([]byte, error) {
	data, err := ioutil.ReadFile("sshgo/private.pem")
	if err != nil {
		return nil, err
	}
	block, _ := pem.Decode(data)
	priv,err := x509.ParsePKCS1PrivateKey(block.Bytes)
	if err != nil {
		return nil, err
	}

	return rsa.DecryptPKCS1v15(rand.Reader,priv,encrypt)
}

func main() {
	data, err := rsaEncrypt("hello golang!")
	if err != nil {
		log.Fatalln(err)
	}
	fmt.Println("rsa encrypt base64:",base64.StdEncoding.EncodeToString(data))

	orig,err := rsaDecrypt(data)
	if err != nil {
		log.Fatalln(err)
	}
	fmt.Println(string(orig))
}

```

### 参考

- [大佬的blog](https://blog.yumaojun.net/2017/02/19/go-crypto/)

- [polaris]([http://blog.studygolang.com/2013/01/go%e5%8a%a0%e5%af%86%e8%a7%a3%e5%af%86rsa%e7%95%aa%e5%a4%96%e7%af%87-%e7%94%9f%e6%88%90rsa%e5%af%86%e9%92%a5/](http://blog.studygolang.com/2013/01/go加密解密rsa番外篇-生成rsa密钥/))