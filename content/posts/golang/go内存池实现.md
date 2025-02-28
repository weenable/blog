---
title: go内存池实现
date: 2025-02-28T18:57:27+08:00
toc: true
readTime: true
autonumber: true
math: true
categories:
  - golang
showTags: false
hideBackToTop: false
---
##### 实现
用于管理固定大小的字节切片（`[]byte`）。内存池的目的在于减少内存分配和垃圾回收的开销，通过重用已经分配的内存块来提高性能
```go
package main

type MemoryPool chan []byte

// 从内存池中返回长度为8的字节切片，如果内存池中没有可用字节切片则新分配
func (l MemoryPool) Borrow() []byte {
	var buf []byte
	select {
	case buf = <-l:
	default:
		buf = make([]byte, 8)
	}

	return buf[:8]
}

// 将字节切片放回内存池
func (l MemoryPool) Return(buf []byte) {
	select {
	case l <- buf:
	default:
		// 垃圾回收
	}
}

func main() {
	var mp MemoryPool = make(chan []byte, 1024)
	sl := mp.Borrow()
	mp.Return(sl)
}

```