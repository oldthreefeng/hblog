---
title: "Go 开发工具安装"
date: 2018-12-09 15:56:10
tags: [goland]
categories: [tools] 
---

[TOC]

## InteliJ IDEA安装

网上的InteliJ IDEA破解安装这里便不赘述了,随便百度了一下，推荐一下这个[blog](https://blog.csdn.net/fly910905/article/details/79875347)
正版安装详细见官网[https://www.jetbrains.com/idea/](https://www.jetbrains.com/idea/)
安装好IDEA后，settings里面plugin安装go插件。Settings ---> pulgins ---> Browse repositories ---> 搜索go，安装`star`最多的即可

## go安装

下载最新的go版本安装[https://golang.org/dl/](https://golang.org/dl/),win10下载 Installer	Windows	x86-64。
开始golang吧！

## goland

~~idea专门为go开发的一款开发工具蛮好用的~,破解工具链接: https://pan.baidu.com/s/17gPPZpekcscthSzMhd7crQ 提取码: g982,~~

```
在安装的bin目录下找到goland.exe.vmoptions和goland64.exe.vmoptions.
-javaagent:C:\Program Files\JetBrains\GoLand 2019.1.3\bin\JetbrainsCrack-release-enc.jar
```

```
ZKVVPH4MIO-eyJsaWNlbnNlSWQiOiJaS1ZWUEg0TUlPIiwibGljZW5zZWVOYW1lIjoi5o6I5p2D5Luj55CG5ZWGIGh0dHA6Ly9pZGVhLmhrLmNuIiwiYXNzaWduZWVOYW1lIjoiIiwiYXNzaWduZWVFbWFpbCI6IiIsImxpY2Vuc2VSZXN0cmljdGlvbiI6IiIsImNoZWNrQ29uY3VycmVudFVzZSI6ZmFsc2UsInByb2R1Y3RzIjpbeyJjb2RlIjoiSUkiLCJmYWxsYmFja0RhdGUiOiIyMDE5LTA3LTAxIiwicGFpZFVwVG8iOiIyMDIwLTA2LTMwIn0seyJjb2RlIjoiQUMiLCJmYWxsYmFja0RhdGUiOiIyMDE5LTA3LTAxIiwicGFpZFVwVG8iOiIyMDIwLTA2LTMwIn0seyJjb2RlIjoiRFBOIiwiZmFsbGJhY2tEYXRlIjoiMjAxOS0wNy0wMSIsInBhaWRVcFRvIjoiMjAyMC0wNi0zMCJ9LHsiY29kZSI6IlBTIiwiZmFsbGJhY2tEYXRlIjoiMjAxOS0wNy0wMSIsInBhaWRVcFRvIjoiMjAyMC0wNi0zMCJ9LHsiY29kZSI6IkdPIiwiZmFsbGJhY2tEYXRlIjoiMjAxOS0wNy0wMSIsInBhaWRVcFRvIjoiMjAyMC0wNi0zMCJ9LHsiY29kZSI6IkRNIiwiZmFsbGJhY2tEYXRlIjoiMjAxOS0wNy0wMSIsInBhaWRVcFRvIjoiMjAyMC0wNi0zMCJ9LHsiY29kZSI6IkNMIiwiZmFsbGJhY2tEYXRlIjoiMjAxOS0wNy0wMSIsInBhaWRVcFRvIjoiMjAyMC0wNi0zMCJ9LHsiY29kZSI6IlJTMCIsImZhbGxiYWNrRGF0ZSI6IjIwMTktMDctMDEiLCJwYWlkVXBUbyI6IjIwMjAtMDYtMzAifSx7ImNvZGUiOiJSQyIsImZhbGxiYWNrRGF0ZSI6IjIwMTktMDctMDEiLCJwYWlkVXBUbyI6IjIwMjAtMDYtMzAifSx7ImNvZGUiOiJSRCIsImZhbGxiYWNrRGF0ZSI6IjIwMTktMDctMDEiLCJwYWlkVXBUbyI6IjIwMjAtMDYtMzAifSx7ImNvZGUiOiJQQyIsImZhbGxiYWNrRGF0ZSI6IjIwMTktMDctMDEiLCJwYWlkVXBUbyI6IjIwMjAtMDYtMzAifSx7ImNvZGUiOiJSTSIsImZhbGxiYWNrRGF0ZSI6IjIwMTktMDctMDEiLCJwYWlkVXBUbyI6IjIwMjAtMDYtMzAifSx7ImNvZGUiOiJXUyIsImZhbGxiYWNrRGF0ZSI6IjIwMTktMDctMDEiLCJwYWlkVXBUbyI6IjIwMjAtMDYtMzAifSx7ImNvZGUiOiJEQiIsImZhbGxiYWNrRGF0ZSI6IjIwMTktMDctMDEiLCJwYWlkVXBUbyI6IjIwMjAtMDYtMzAifSx7ImNvZGUiOiJEQyIsImZhbGxiYWNrRGF0ZSI6IjIwMTktMDctMDEiLCJwYWlkVXBUbyI6IjIwMjAtMDYtMzAifSx7ImNvZGUiOiJSU1UiLCJmYWxsYmFja0RhdGUiOiIyMDE5LTA3LTAxIiwicGFpZFVwVG8iOiIyMDIwLTA2LTMwIn1dLCJoYXNoIjoiMTM1NTgzMjIvMCIsImdyYWNlUGVyaW9kRGF5cyI6NywiYXV0b1Byb2xvbmdhdGVkIjpmYWxzZSwiaXNBdXRvUHJvbG9uZ2F0ZWQiOmZhbHNlfQ==-i/ZK8vfXLX80OFpkhwEo9QxMhsWaOu3SfBmNPup63N0kjM2XBIoR67s8fk0Li45CreS2zQcPZdypLPeyRrdrUYGTw77tkK/kUygxEwRKauqgdJhUs+881TGitcmZvk8obLXjjpv+tZEbV31ee6Fb2/iuK36Q1NCuhKGlo8mA68kGXLOk5ppRYCqQUnHY2zk8spzxC/yJtG+JAQGlPDyvQmkQ5taRxM77b1/v2/62t5Xa2HqnPkuJBrS+XXuGz++RBuYEv6cVe5hmsUaQJZe9/Z4BrhMy48fVEG6bsKTmJ4yILs9sSyUM6uA05AOm8lXWmCG3m9AdVyawsWqBJIn7Rw==-MIIElTCCAn2gAwIBAgIBCTANBgkqhkiG9w0BAQsFADAYMRYwFAYDVQQDDA1KZXRQcm9maWxlIENBMB4XDTE4MTEwMTEyMjk0NloXDTIwMTEwMjEyMjk0NlowaDELMAkGA1UEBhMCQ1oxDjAMBgNVBAgMBU51c2xlMQ8wDQYDVQQHDAZQcmFndWUxGTAXBgNVBAoMEEpldEJyYWlucyBzLnIuby4xHTAbBgNVBAMMFHByb2QzeS1mcm9tLTIwMTgxMTAxMIIBIjANBgkqhkiG9w0BAQEFAAOCAQ8AMIIBCgKCAQEAxcQkq+zdxlR2mmRYBPzGbUNdMN6OaXiXzxIWtMEkrJMO/5oUfQJbLLuMSMK0QHFmaI37WShyxZcfRCidwXjot4zmNBKnlyHodDij/78TmVqFl8nOeD5+07B8VEaIu7c3E1N+e1doC6wht4I4+IEmtsPAdoaj5WCQVQbrI8KeT8M9VcBIWX7fD0fhexfg3ZRt0xqwMcXGNp3DdJHiO0rCdU+Itv7EmtnSVq9jBG1usMSFvMowR25mju2JcPFp1+I4ZI+FqgR8gyG8oiNDyNEoAbsR3lOpI7grUYSvkB/xVy/VoklPCK2h0f0GJxFjnye8NT1PAywoyl7RmiAVRE/EKwIDAQABo4GZMIGWMAkGA1UdEwQCMAAwHQYDVR0OBBYEFGEpG9oZGcfLMGNBkY7SgHiMGgTcMEgGA1UdIwRBMD+AFKOetkhnQhI2Qb1t4Lm0oFKLl/GzoRykGjAYMRYwFAYDVQQDDA1KZXRQcm9maWxlIENBggkA0myxg7KDeeEwEwYDVR0lBAwwCgYIKwYBBQUHAwEwCwYDVR0PBAQDAgWgMA0GCSqGSIb3DQEBCwUAA4ICAQAF8uc+YJOHHwOFcPzmbjcxNDuGoOUIP+2h1R75Lecswb7ru2LWWSUMtXVKQzChLNPn/72W0k+oI056tgiwuG7M49LXp4zQVlQnFmWU1wwGvVhq5R63Rpjx1zjGUhcXgayu7+9zMUW596Lbomsg8qVve6euqsrFicYkIIuUu4zYPndJwfe0YkS5nY72SHnNdbPhEnN8wcB2Kz+OIG0lih3yz5EqFhld03bGp222ZQCIghCTVL6QBNadGsiN/lWLl4JdR3lJkZzlpFdiHijoVRdWeSWqM4y0t23c92HXKrgppoSV18XMxrWVdoSM3nuMHwxGhFyde05OdDtLpCv+jlWf5REAHHA201pAU6bJSZINyHDUTB+Beo28rRXSwSh3OUIvYwKNVeoBY+KwOJ7WnuTCUq1meE6GkKc4D/cXmgpOyW/1SmBz3XjVIi/zprZ0zf3qH5mkphtg6ksjKgKjmx1cXfZAAX6wcDBNaCL+Ortep1Dh8xDUbqbBVNBL4jbiL3i3xsfNiyJgaZ5sX7i8tmStEpLbPwvHcByuf59qJhV/bZOl8KqJBETCDJcY6O2aqhTUy+9x93ThKs1GKrRPePrWPluud7ttlgtRveit/pcBrnQcXOl1rHq7ByB8CFAxNotRUYL9IF5n3wJOgkPojMy6jetQA5Ogc8Sm7RG6vg1yow==
```

~~激活码由<https://blog.csdn.net/qq_41570658/article/details/85708172>大佬提供~~

