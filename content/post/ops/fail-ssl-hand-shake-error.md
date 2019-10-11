---
title: "小程序request:fail ssl hand shake error"
date: "2019-04-22 09:59:32"
tags: [linux,ssl]
categories: [server]
---

[TOC]

>域名已经备案，证书全部都有，也在后台配置了，但是安卓手机预览，还是请求失败， PC端和iphone端是可以请求数据出来的新版开发者工具增加了https检查功能；可使用此功能直接检查排查ssl协议版本问题： 
>报错`request:fail ssl hand shake error`。

**问题来源**：
>测试环境的域名更改，区别于正式环境，进行了如下的变更。
```
test_xiaowugui_video.jinxianghuangjin.com<--- xiaogui3.jinxianghuangjin.com
test_gl.jinxianghuangjin.com<-------- gl.jinxianghuangjin.com
test_xiaowugui.jinxianghuangjin.com<----xiaowugui.jinxianghuangjin.com
```
>改变域名之后，网上访问正常，没有任何毛病。开发火急火燎的跑过来问我说是不是ssl证书失效了，小程序不行了，报错了。一脸懵，应该不会报错的啊，我这边访问一切正常，然后看到开发的程序log，有以上的报错，便有了这篇记录。
### 解决思路
**方案**
第一方案是缺少中间证书。详情可以看看这个网址[微信小程序访问提示：request:failsslhandshakeerror](http://blog.sina.com.cn/s/blog_4c4daf740102xdeo.html).为了解决这个问题，重新换了新的ssl证书。但问题依旧没有解决。
**更换域名**
想到了域名更换引发的问题，便怀疑是不是域名有下划线的问题，然后去[苹果ATS监测](https://cloud.tencent.com/product/ssl#)检测，发现域名检测不通，瞬间意识到了问题。更改域名即可解决问题。算是踩了一个小坑吧~
```
test-xiaowugui-video.jinxianghuangjin.com<--- xiaogui3.jinxianghuangjin.com
test-gl.jinxianghuangjin.com<-------- gl.jinxianghuangjin.com
test-xiaowugui.jinxianghuangjin.com<----xiaowugui.jinxianghuangjin.com
```
更换域名后，立马ATS检查通过。

### 参考
>证书常见问题：参考[微信官方文档](https://pay.weixin.qq.com/wiki/doc/api/jsapi.php?chapter=10_4)
>域名监测ATS：参考[苹果ATS监测](https://cloud.tencent.com/product/ssl#)
>同时测试ios和安卓，假如有一方可以，一方不行，则是证书问题，请选用受认可的证书
