---
title: "go filemode 源码理解"
date: 2019-08-26 14:56:10
tags: [go,mode]
categories: [golang] 
---

[TOC]

## `FileMode`

官方的解释:

>A `FileMode` represents a file's mode and permission bits.
>The bits have the same definition on all systems, so that information about files can be moved from one system to another portably. Not all bits apply to all systems.
>The only required bit is `ModeDir` for directories.

意思大概: `FileMode` 代表了一个文件的属性和权限位,这些权限位在所有的文件系统都一样;因此在各个系统中都能很好的迁移,唯一要求的位就是`ModeDir`.

```
type FileMode uint32

const (
	// The single letters are the abbreviations
	// used by the String method's formatting.
	ModeDir        FileMode = 1 << (32 - 1 - iota) // d: is a directory
	ModeAppend                                     // a: append-only
	ModeExclusive                                  // l: exclusive use
	ModeTemporary                                  // T: temporary file; Plan 9 only
	ModeSymlink                                    // L: symbolic link
	ModeDevice                                     // D: device file
	ModeNamedPipe                                  // p: named pipe (FIFO)
	ModeSocket                                     // S: Unix domain socket
	ModeSetuid                                     // u: setuid
	ModeSetgid                                     // g: setgid
	ModeCharDevice                                 // c: Unix character device, when ModeDevice is set
	ModeSticky                                     // t: sticky
	ModeIrregular                                  // ?: non-regular file; nothing else is known about this file

	// Mask for the type bits. For regular files, none will be set.
	ModeType = ModeDir | ModeSymlink | ModeNamedPipe | ModeSocket | ModeDevice | ModeCharDevice | ModeIrregular

	ModePerm FileMode = 0777 // Unix permission bits
)
```

官方用`uint32`来存储定义每个一个文件的`mode`;其中引用到的位只有`uint32`的前13位;具体可以看后续的代码分析.`uint32`的后9位来存储文件的权限`permissions`.为了更清楚的表示.我这边验证了一下.

```go
package main

import (
	"fmt"
	"github.com/imroc/biu"   
    //这里引用了一下大牛的binary包,需要自己手动添加一下os.FileMode类型在函数里面.
    //在biu.go里添加一下即可,不然会无法解析,因为将os.FileMode转uint32类型进行匹配.
    //case os.FileMode:
	//	  s = Uint32ToBinaryString(uint32(v))
	"os"
)

func main(){
    //用了一下biu的转二进制的包,这样看起来更清晰,
    //如果直接用fmt.Printf("%b",os.ModePerm)前面的0会被省略.
	fmt.Println(biu.ToBinaryString(os.ModePerm))
	fmt.Println(biu.ToBinaryString(os.ModeDir))
	fmt.Println(biu.ToBinaryString(os.ModeAppend))
	fmt.Println(biu.ToBinaryString(os.ModeExclusive))
	fmt.Println(biu.ToBinaryString(os.ModeTemporary))
	fmt.Println(biu.ToBinaryString(os.ModeSymlink))
	fmt.Println(biu.ToBinaryString(os.ModeDevice))
	fmt.Println(biu.ToBinaryString(os.ModeNamedPipe))
	fmt.Println(biu.ToBinaryString(os.ModeSocket))
	fmt.Println(biu.ToBinaryString(os.ModeSetuid))
	fmt.Println(biu.ToBinaryString(os.ModeSetgid))
	fmt.Println(biu.ToBinaryString(os.ModeCharDevice))
	fmt.Println(biu.ToBinaryString(os.ModeSticky))
	fmt.Println(biu.ToBinaryString(os.ModeIrregular))
	fmt.Println(biu.ToBinaryString(os.ModeType))
}
/*
[00000000 00000000 00000001 11111111]
[10000000 00000000 00000000 00000000]
[01000000 00000000 00000000 00000000]
[00100000 00000000 00000000 00000000]
[00010000 00000000 00000000 00000000]
[00001000 00000000 00000000 00000000]
[00000100 00000000 00000000 00000000]
[00000010 00000000 00000000 00000000]
[00000001 00000000 00000000 00000000]
[00000000 10000000 00000000 00000000]
[00000000 01000000 00000000 00000000]
[00000000 00100000 00000000 00000000]
[00000000 00010000 00000000 00000000]
[00000000 00001000 00000000 00000000]
[10001111 00101000 00000000 00000000]
*/	
```

因为经常使用`Linux`系统,对其中的文件系统的权限设置很敢兴趣,便研究了`drwxrwxrwx`这个文件夹`0777`权限在go中是如何实现转换的.源码如下

```go
func (m FileMode) String() string {
	const str = "dalTLDpSugct?"
	var buf [32]byte // Mode is uint32.
	w := 0
	for i, c := range str {
		if m&(1<<uint(32-1-i)) != 0 {
			buf[w] = byte(c)
			w++
		}
	}
	if w == 0 {
		buf[w] = '-'
		w++
	}
	const rwx = "rwxrwxrwx"
	for i, c := range rwx {
		if m&(1<<uint(9-1-i)) != 0 {
			buf[w] = byte(c)
		} else {
			buf[w] = '-'
		}
		w++
	}
	return string(buf[:w])
}
```

源码很简洁,一个`String()`函数,就将对应关系弄的明明白白.前面的定义是处理`FileMode`的13种类型,占`String()`的一个字节例如`d---------`;后面定义的是`FileMode`的权限也就是`rwxrwxrwx`用`uint32`表示即为`0777`.解读如下:

```go
	const str = "dalTLDpSugct?" 
//这个对应了上面DirMode的13个类型,也就是FileMode这个`uint32`类型的前13位bit.
	var buf [32]byte  // 用来存储uint32位的每一位的结果,存储类型为byte
//接着就进行了遍历。对str遍历。
	for i, c := range str {
		if m&(1<<uint(32-1-i)) != 0 {
			buf[w] = byte(c)
			w++
		}
	}
// m&(1<<uint(32-1-i)) 这个是将1左移uint(32-1-i)位,在和m进行逻辑与
// 相当于m和FileMode的13种类型进行比对,如果m和FileMode的13种类型有匹配关系的.
// 那么与结果看到不为0,必定为1.这个确定的是文件的mode类型,
// 如果m是文件夹,那么m对应的String()也就是`d---------` .

// permissions的函数解决如下.
	const rwx = "rwxrwxrwx" //和unix的权限类型对应起来了.看起来很熟悉对不对
	for i, c := range rwx {
        if m&(1<<uint(9-1-i)) != 0 {
        	// 如果 & 结果不为零; 说明为真 ; 将该位置为相应的字符;否则置空
			// 第一位即最高位为1,可读权限 1 0000 0000  = 400
			// 第二位为1               0 1000 0000  = 200
			// 第三位为1               0 0100 0000  = 100
			// rwx ==> 700
			// 第四位为1               0 0010 0000  = 040
			// 第五位为1               0 0001 0000  = 020
			// 第六位为1               0 0000 1000  = 010
			// rwx ==> 070
			// 第七位为1               0 0000 0100  = 004
			// 第八位为1               0 0000 0010  = 002
			// 第九位为1               0 0000 0001  = 001
			buf[w] = byte(c)
		} else {
            // 如果&结果为零,说明该位为0,就直接置空'-'
			buf[w] = '-'
		}
		w++
	}
```

检验一下:

```go
package main

import (
	"fmt"
	"github.com/imroc/biu"  
	"os"
)

func main(){
	var m1 os.FileMode
	var m2 uint16  //权限控制其实uint16位就足够了,但是加上`mode`的13种类型就不够了
	m2 = 0775
	fmt.Printf("%b\n",m2)
	m1 = 0775 //每一位都是8进制, 775 -> 111111101
	fmt.Printf("%b\n",m1)
	fmt.Println(m1.String())
	fmt.Println(os.ModeDir.String())
}
/*
111111101
111111101
-rwxrwxr-x
d---------
*/
```

记录一下学习`golang`的过程~第34天