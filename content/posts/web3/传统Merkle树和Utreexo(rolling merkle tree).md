---
title: 传统merkle树和utreexo(rolling merkle tree)
date: 2025-05-06T19:14:03+08:00
toc: true
readTime: true
autonumber: true
math: true
categories:
  - web3
showTags: false
hideBackToTop: false
---
### 传统Merkle树实现
```
package main  
  
import (  
    "crypto/sha256"  
    "encoding/hex"
    "fmt"
)  
  
func hashPair(a, b []byte) []byte {  
    h := sha256.New()  
    h.Write([]byte{0x01}) // 内部节点前缀  
    if hex.EncodeToString(a) < hex.EncodeToString(b) {  
       h.Write(a)  
       h.Write(b)  
    } else {  
       h.Write(b)  
       h.Write(a)  
    }  
    return h.Sum(nil)  
}  
  
type MerkleTree struct {  
    nodes [][]byte  
}  
  
func NewMerkleTree() *MerkleTree {  
    return &MerkleTree{nodes: make([][]byte, 0)}  
}  
  
func (mt *MerkleTree) Add(data []byte) {  
    mt.nodes = append(mt.nodes, data)  
}  
  
func (mt *MerkleTree) Root() []byte {  
    if len(mt.nodes) == 0 {  
       return nil  
    }  
    level := make([][]byte, len(mt.nodes))  
    copy(level, mt.nodes)  
  
    for len(level) > 1 {  
       var nextLevel [][]byte  
       for i := 0; i < len(level); i += 2 {  
          if i+1 >= len(level) {  
             nextLevel = append(nextLevel, hashPair(level[i], level[i]))  
          } else {  
             nextLevel = append(nextLevel, hashPair(level[i], level[i+1]))  
          }  
       }  
       level = nextLevel  
    }  
  
    return level[0]  
}  
  
func main() {  
    testData := [][]byte{  
       []byte("Transaction1"),  
       []byte("Transaction2"),  
       []byte("Transaction3"),  
       []byte("Transaction4"),  
    }  
    fmt.Println("=== 传统Merkle树测试 ===")  
    mt := NewMerkleTree()  
    for i, data := range testData {  
       mt.Add(data)  
       fmt.Printf("添加第%d个交易后的根哈希: %x\n", i+1, mt.Root())  
    }  
}


=== 传统Merkle树测试 ===
添加第1个交易后的根哈希: 5472616e73616374696f6e31
添加第2个交易后的根哈希: ae8f5f113879432c68c9aa01a45e2cf0b3da92243277f0db78cf3a868141f73e
添加第3个交易后的根哈希: a76764553968d075e0a82751ec1dbced37b5be3ef29bb3765867da9a5178e81d
添加第4个交易后的根哈希: 170d56828552760c53ae90f4241b6fdd8a2d85c783264e92d887557527802169
```

### UTreexo（rolling merkle tree）实现
和传统merkle相比，主要差异点在于add节点的时候，rolling merkle只需要存储最终的合并根哈希
```
type RollingMerkle struct {  
    roots     [][]byte  
    numLeaves int  
}  
  
func NewRollingMerkle() *RollingMerkle {  
    return &RollingMerkle{roots: make([][]byte, 0)}  
}  
  
func (rm *RollingMerkle) Add(data []byte) {  
    newRoot := hashLeaf(data)  
    h := 0  
    for (rm.numLeaves>>h)&1 == 1 {  
       // 弹出最后一个根并合并  
       lastRoot := rm.roots[len(rm.roots)-1]  
       rm.roots = rm.roots[:len(rm.roots)-1]  
       newRoot = hashPair(lastRoot, newRoot)  
       h++  
    }  
    rm.roots = append(rm.roots, newRoot)  
    rm.numLeaves++  
}  
  
func (rm *RollingMerkle) Root() []byte {  
    if len(rm.roots) == 0 {  
       return nil  
    }  
    // 合并所有剩余根节点  
    finalRoot := rm.roots[0]  
    for _, root := range rm.roots[1:] {  
       finalRoot = hashPair(finalRoot, root)  
    }  
    return finalRoot  
}  
  
// ========================== 哈希工具函数 ==========================func hashLeaf(data []byte) []byte {  
    h := sha256.New()  
    h.Write([]byte{0x00}) // 叶子节点前缀  
    h.Write(data)  
    return h.Sum(nil)  
}  
  
func hashPair(a, b []byte) []byte {  
    h := sha256.New()  
    h.Write([]byte{0x01}) // 内部节点前缀  
    if hex.EncodeToString(a) < hex.EncodeToString(b) {  
       h.Write(a)  
       h.Write(b)  
    } else {  
       h.Write(b)  
       h.Write(a)  
    }  
    return h.Sum(nil)  
}  
  
// ========================== 测试代码 ==========================func main() {  
    testData := [][]byte{  
       []byte("Transaction1"),  
       []byte("Transaction2"),  
       []byte("Transaction3"),  
       []byte("Transaction4"),  
    }  
      
    fmt.Println("\n=== Rolling Merkle测试 ===")  
    rm := NewRollingMerkle()  
    for i, data := range testData {  
       rm.Add(data)  
       fmt.Printf("添加第%d个交易后的根哈希: %x\n", i+1, rm.Root())  
       fmt.Printf("当前根数量: %d\n", len(rm.roots))  
    }  
}

```