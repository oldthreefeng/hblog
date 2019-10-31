---
title: "LRU 算法"
date: 2019-10-12 12:40:10
tags: [lru,doublyLinkedList,hashMap]
categories: [algorithm,golang] 
---

## 操作系统内存的管理

> 关于操作系统内存管理，如何节省利用容量不大的内存多的进程提供资源。而内存的虚拟存储管理是现在最通用，最成功的方式。在内存有限的情况下，扩展一部分外存作为虚拟内存，真正的内存只存储当前运行时所用到的信息，极大的扩充了内存的功能，极大提高计算机的并发度，虚拟页式存储管理，则是将进程所需空间划分为多个页面，内存中只存放当前所需页面，将其余页面放入外存的管理方式.

> 虚拟页式存储管理增加了进程所需的内存空间，却也带来了运行时间变长这一缺点，运行过程中，需要将外存中存放的信息和内存中已有的进行交换，由于外存的低速，影响了运行时间因此，应当采取尽量好的算法以减少读取外存的次数.

> 对于虚拟页式存储，内外存信息替换是以页面为单位进行的，当需要一个外存中的页面是 将它调入内存，同时为了保持原有空间大小，我们需要不断地剔除掉一些存页面。数据块大小有限，因此我们需要每次调用外存数据时，都能准确的命中。这时就需要一个较好的页面管理算法.

## LRU
LRU是Least Recently Used的缩写，即最近最少使用，是一种常用的页面置换算法，选择最近最久未使用的页面予以淘汰。该算法赋予每个页面一个访问字段，用来记录一个页面自上次被访问以来所经历的时间 t，当须淘汰一个页面时，选择现有页面中其 t 值最大的，即最近最少使用的页面予以淘汰。

### LRU缓存机制

设计和实现一个 ` LRU` (最近最少使用) 缓存机制。它应该支持以下操作： 获取数据 `get `和 写入数据` put `。

获取数据` get(key)` - 如果密钥 (key) 存在于缓存中，则获取密钥的值（总是正数），否则返回 -1。
写入数据 `put(key, value)` - 如果密钥不存在，则写入其数据值。当缓存容量达到上限时，它应该在写入新数据之前删除最近最少使用的数据值，从而为新的数据值留出空间。

示例:

```
LRUCache cache = new LRUCache( 2 /* 缓存容量 */ );

cache.put(1, 1);
cache.put(2, 2);
cache.get(1);       // 返回  1
cache.put(3, 3);    // 该操作会使得密钥 2 作废
cache.get(2);       // 返回 -1 (未找到)
cache.put(4, 4);    // 该操作会使得密钥 1 作废
cache.get(1);       // 返回 -1 (未找到)
cache.get(3);       // 返回  3
cache.get(4);       // 返回  4
```
- 思路

这个问题可以用哈希表，辅以双向链表记录键值对的信息。所以可以在 `O(1)` 时间内完成 `put` 和 `get` 操作，同时也支持 `O(1)` 删除第一个添加的节点。
![lruCache](http://pic.fenghong.tech/lrucache.png)
如果key存在缓存中, 则获取key的value
每次数据项被查询到时，都将此数据项移动到链表头部（O(1)的时间复杂度）
这样，在进行过多次查找操作后，最近被使用过的内容就向链表的头移动，而没有被使用的内容就向链表的后面移动
当需要替换时，链表最后的位置就是最近最少被使用的数据项，我们只需要将最新的数据项放在链表头部，
当Cache满时，淘汰链表最后的位置就是了。

```go
/*
@Time : 2019/10/12 10:04
@Author : louis
@File : 146-lrucache
@Software: GoLand
*/

package leetcode

import "container/list"

// 内存缓存LRU算法: 最近最少使用

type LRUCache struct {
	cap int                   // capacity
	l   *list.List            // doubly linked list
	m   map[int]*list.Element // hash table for checking if list node exists
}

type pair struct {
	key   int
	value int
}

func NewLRU(capacity int) LRUCache {
	return LRUCache{
		cap: capacity,
		l:   new(list.List),
		m:   make(map[int]*list.Element, capacity),
	}
}

// 如果key存在缓存中, 则获取key的value
// 每次数据项被查询到时，都将此数据项移动到链表头部（O(1)的时间复杂度）
// 这样，在进行过多次查找操作后，最近被使用过的内容就向链表的头移动，而没有被使用的内容就向链表的后面移动
// 当需要替换时，链表最后的位置就是最近最少被使用的数据项，我们只需要将最新的数据项放在链表头部，
// 当Cache满时，淘汰链表最后的位置就是了。
func (lru *LRUCache) Get(key int) int {
	if node, ok := lru.m[key]; ok {
		val := node.Value.(*list.Element).Value.(pair).value
		//move the node to front
		lru.l.MoveToFront(node)
		return val
	}
	return -1
}

// 写入数据, 如果key不存在, 则写入其数据值, 当缓存容量达到上限时,
// 它应该在写入新数据之前删除最近最少使用的value.
func (lru *LRUCache) Put(key, value int) {
	if node, ok := lru.m[key]; ok {
		// 存在,则直接移动node到链表首部,并更新value
		lru.l.MoveToFront(node)
		node.Value.(*list.Element).Value = pair{key: key, value: value}

	} else {
		// list 满了之后 ; 删除链表的最后一个节点
		if lru.l.Len() == lru.cap {
			delIndex := lru.l.Back().Value.(*list.Element).Value.(pair).key
			// 删除hashMap里面的最后的元素
			delete(lru.m, delIndex)
			// 删除list最后一个节点
			lru.l.Remove(lru.l.Back())
		}

		// 初始化node节点
		node := &list.Element{
			Value: pair{
				key:   key,
				value: value,
			},
		}
		// 将新的node节点放入list
		p := lru.l.PushFront(node)
		lru.m[key] = p
	}
}

```
### 测试用例

```cgo
package leetcode

import (
	"fmt"
	"testing"
)

func TestConstructorLRU(t *testing.T) {
	var cache = NewLRU(2)
	cache.Put(1,1)
	cache.Put(2,2)
	fmt.Println(cache.Get(1))
	cache.Put(3,3)
	fmt.Println(cache.Get(2))
	fmt.Println(cache.Get(3))
	cache.Put(4,4)
	fmt.Println(cache.Get(1))
	fmt.Println(cache.Get(3))
	fmt.Println(cache.Get(4))
	//1 -1 3 -1 3 4
}
```
