---
title: 哈希表渐进式Rehash实现
date: 2025-02-28T19:06:04+08:00
toc: true
readTime: true
autonumber: true
math: true
categories:
  - 算法数据结构
showTags: false
hideBackToTop: false
---
##### 说明
1. 哈希冲突采用链地址法
2. 在操作中进行渐进式扩容迁移

##### 代码实现

```
package main

// 初始哈希表大小
const initialSize = 4

// 哈希表元素
type entry struct {
	key   string
	value interface{}
	next  *entry
}

type dict struct {
	ht          [2][]*entry
	rehashIndex int
}

func NewDict() *dict {
	d := &dict{
		ht:          [2][]*entry{make([]*entry, initialSize), nil},
		rehashIndex: -1,
	}

	return d
}

// 实现一个哈希函数
func (d *dict) hash(key string) int {
	h := 0
	for i := 0; i < len(key); i++ {
		h = 31*h + int(key[i])
	}

	return h
}

// 插入元素
func (d *dict) Add(key string, value interface{}) {
	if d.rehashIndex != -1 {
		d.rehash()
	}

	// 如果此刻没有在渐进扩容期间，并且当前元素个数>=容量一半，触发渐进扩容
	if d.ht[1] == nil && len(d.ht[0]) > 0 && len(d.ht[0]) >= cap(d.ht[0])/2 {
		d.ht[1] = make([]*entry, len(d.ht[0])*2) //扩容一倍
		d.rehashIndex = 0
	}

	index := d.hash(key) % len(d.ht[0])
	newEntry := &entry{key: key, value: value, next: d.ht[0][index]}
	d.ht[0][index] = newEntry
}

// 获取元素
func (d *dict) Get(key string) (interface{}, bool) {
	if d.rehashIndex != -1 {
		d.rehash()
	}

	// 在h[0]中找
	index := d.hash(key) % len(d.ht[0])
	for curEntry := d.ht[0][index]; curEntry != nil; curEntry = curEntry.next {
		if curEntry.key == key {
			return curEntry.value, true
		}
	}

	// 在h[1]中找，已经迁移
	if d.ht[1] != nil {
		index = d.hash(key) % len(d.ht[1])
		for entry := d.ht[1][index]; entry != nil; entry = entry.next {
			if entry.key == key {
				return entry.value, true
			}
		}
	}

	return nil, false
}

// 渐进rehash执行的函数
func (d *dict) rehash() {
	if d.rehashIndex == -1 {
		return
	}

	// 渐进移动一个桶的数据
	for i := 0; i < 1 && d.rehashIndex < len(d.ht[0]); i++ {
		if d.ht[0][d.rehashIndex] != nil {
			curEntry := d.ht[0][d.rehashIndex]
			for curEntry != nil {
				nextEntry := curEntry.next
				index := d.hash(curEntry.key) % len(d.ht[1])
				curEntry.next = d.ht[1][index]
				d.ht[1][index] = curEntry
				curEntry = nextEntry
			}

			d.ht[0][d.rehashIndex] = nil
		}
		d.rehashIndex++
	}

	if d.rehashIndex >= len(d.ht[0]) {
		d.ht[0] = d.ht[1]
		d.ht[1] = nil
		d.rehashIndex = -1
	}
}


func main() {
	d := rehash.NewDict()
	d.Add("foo", "bar")
	d.Add("baz", "qux")

	val, found := d.Get("foo")
	if found {
		fmt.Println("Found foo:", val)
	} else {
		fmt.Println("foo not found")
	}

	val, found = d.Get("baz")
	if found {
		fmt.Println("Found baz:", val)
	} else {
		fmt.Println("baz not found")
	}

	// 加这两个元素前已经触发扩容，因为设定元素个数>=容量一半则扩容
	d.Add("new", "value")
	d.Add("another", "entry")

	val, found = d.Get("new")
	if found {
		fmt.Println("Found new:", val)
	} else {
		fmt.Println("new not found")
	}

}

```