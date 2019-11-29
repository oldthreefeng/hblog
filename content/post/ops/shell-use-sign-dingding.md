---
title: "使用shell脚本发送钉钉通知"
date: 2019-11-29T13:54:18+08:00
tags: [ops,jenkins,dingding]
categories: [ops,jenkins]
---

### 使用自定义机器人注意事项

- **获取到Webhook地址后，用户可以向这个地址发起HTTP POST 请求，即可实现给该钉钉群发送消息。注意，发起POST请求时，必须将字符集编码设置成UTF-8。

- **当前自定义机器人支持文本 (text)、链接 (link)、markdown、ActionCard、FeedCard消息类型**，大家可以根据自己的使用场景选择合适的消息类型，达到最好的展示样式。

- **自定义机器人发送消息时，可以通过手机号码指定“被@人列表”。在“被@人列表”里面的人员收到该消息时，会有@消息提醒(免打扰会话仍然通知提醒，首屏出现“有人@你”)。

- **当前机器人尚不支持应答机制** (该机制指的是群里成员在聊天@机器人的时候，钉钉回调指定的服务地址，即Outgoing机器人)。
-  每个机器人每分钟最多发送20条。消息发送太频繁会严重影响群成员的使用体验，大量发消息的场景 (譬如系统监控报警) 可以将这些信息进行整合，通过markdown消息以摘要的形式发送到群里。 

## 安全设置

安全设置目前有3种方式：

**（1）方式一，自定义关键词**

最多可以设置10个关键词，消息中至少包含其中1个关键词才可以发送成功。

例如：添加了一个自定义关键词：监控报警

则这个机器人所发送的消息，必须包含 监控报警 这个词，才能发送成功。



**（2）方式二，加签**

第一步，把timestamp+"\n"+密钥当做签名字符串，使用HmacSHA256算法计算签名，然后进行Base64 encode，最后再把签名参数再进行urlEncode，得到最终的签名（需要使用UTF-8字符集）。

这里主要演示加签方法.

### 加签shell脚本发送版本

shell脚本发送钉钉通知, 这是调用了阿里云api和python脚本, 实现自动发送阿里云可用余额给运维群. 

```bash
#!/usr/bin/env bash

## author: louis@wangke.co

function notify(){
    curl "https://oapi.dingtalk.com/robot/send?access_token=$dingdingtoken&timestamp=$timestamp&sign=$sign"    -H 'Content-Type: application/json'    -d "{'msgtype': 'markdown',
        'markdown': {
            'title': '阿里云费用余额',
            'text': '##  阿里云账户 \n### 可用现金余额: $AvaliableCash\n### 可用余额: $AvaliableMount\n### 查询时间: $DateStamp'
        }
    }"
}
dingdingtoken=xxxxxxxx
getkey=$(python a.py)
timestamp=${getkey:0:13}
sign=$(echo "${getkey:13:100}" | tr  -d '\n')
DateStamp=$(date -d @${getkey:0:10} "+%F %H:%m:%S")
AvaliableCash=$(aliyun bssopenapi QueryAccountBalance | jq .Data.AvailableCashAmount)
AvaliableMount=$(aliyun bssopenapi QueryAccountBalance | jq .Data.AvailableAmount)

notify
```

使用`python`获取时间戳及加密`sign`

```python
#python 2.6
import time
import hmac
import hashlib
import base64
import urllib

timestamp = long(round(time.time() * 1000))
secret = 'secret'
secret_enc = bytes(secret).encode('utf-8')
string_to_sign = '{0}\n{1}'.format(timestamp, secret)
string_to_sign_enc = bytes(string_to_sign).encode('utf-8')
hmac_code = hmac.new(secret_enc, string_to_sign_enc, digestmod=hashlib.sha256).digest()
sign = urllib.quote_plus(base64.b64encode(hmac_code))
print(timestamp)
print(sign)

"""
#python 2.7
import time
import hmac
import hashlib
import base64
import urllib

timestamp = long(round(time.time() * 1000))
secret = 'secret'
secret_enc = bytes(secret).encode('utf-8')
string_to_sign = '{}\n{}'.format(timestamp, secret)
string_to_sign_enc = bytes(string_to_sign).encode('utf-8')
hmac_code = hmac.new(secret_enc, string_to_sign_enc, digestmod=hashlib.sha256).digest()
sign = urllib.quote_plus(base64.b64encode(hmac_code))
print(timestamp)
print(sign)
"""
```
脚本执行

```bash
$ sh notify_aliyun_bss.sh
{"errcode":0,"errmsg":"ok"}
```

最后, 提一下不需要使用加签的方法, 很早之前写过jenkins发送脚本.

```bash
#!/bin/sh
title=$1
messageUrl=$2
picUrl=$4
text=$3
PHONE="158215*****"
TOKEN=$5
DING="curl -H \"Content-Type: application/json\" -X POST --data '{\"msgtype\": \"link\", \"link\": {\"messageUrl\": \"${messageUrl}\", \"title\": \"${title}\", \"picUrl\": \"${picUrl}\", \"text\": \"${text}\",}, \"at\": {\"atMobiles\": [${PHONE}], \"isAtAll\": false}}' ${TOKEN}"
eval $DING
```
脚本执行

```bash
$ sh /var/jenkins_home/dingding.sh jenkins-test-qx-44 \ http://fenghong.tech:8088/job/test-qx/44/ 发布成功 \
http://icons.iconarchive.com/icons/paomedia/small-n-flat/1024/sign-check-icon.png \
https://oapi.dingtalk.com/robot/send?access_token=****
```

### 错误码

```
// 消息内容中不包含任何关键词
{
  "errcode":310000,
  "errmsg":"keywords not in content"
}

// timestamp 无效
{
  "errcode":310000,
  "errmsg":"invalid timestamp"
}

// 签名不匹配
{
  "errcode":310000,
  "errmsg":"sign not match"
}

// IP地址不在白名单
{
  "errcode":310000,
  "errmsg":"ip X.X.X.X not in whitelist"
}
```

### 参考

- [钉钉自定义机器人]( https://ding-doc.dingtalk.com/doc#/serverapi2/qf2nxq )