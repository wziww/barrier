### heap

### <img src="https://upload.wikimedia.org/wikipedia/commons/thumb/7/7d/FullBT_CompleteBT.jpg/400px-FullBT_CompleteBT.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1" alt="img"  />

> 堆：父节点始终大于等于（小于等于）子节点的完全二叉树，根据父节点的相对大小可分为大小堆
>
> 常以数组形式存在，父节点和子节点的下标关系为 n (父节点) n×2+1（左节点） n×2+2（右节点），可见下图 

``` 
[1, 2, 3, 4, 5,  6,   7,  8]
 |__|__|  |  |   |    |   |
    |_____|__|   |    |   |
       |_________|____|   |
          |_______________|
             
                      
                      
1 => 2,3       |  0 => 1,2
2 => 4,5       |  1 => 3,4
3 => 6,7       |  2 => 5,6
4 => 7,8       |  3 => 7,8
....
```

![{\ displaystyle {\ begin {aligned} \ sum _ {h = 0} ^ {\ lfloor \ log n \ rfloor} {\ frac {n} {2 ^ {h}}} O（h）＆= O \ left （n \ sum _ {h = 0} ^ {\ lfloor \ log n \ rfloor} {\ frac {h} {2 ^ {h}}} \ right）\\＆= O \ left（n \ sum _ { h = 0} ^ {\ infty} {\ frac {h} {2 ^ {h}}} \ right）\\＆= O（n）\ end {aligned}}}](https://wikimedia.org/api/rest_v1/media/math/render/svg/2b1f5dac83f79e16b8794611e4b4a91594c422d8)

> 堆构造时间复杂度公式

```go
// file go/src/container/heap/heap.go

// Copyright 2009 The Go Authors. All rights reserved.
// Use of this source code is governed by a BSD-style
// license that can be found in the LICENSE file.

// Package heap provides heap operations for any type that implements
// heap.Interface. A heap is a tree with the property that each node is the
// minimum-valued node in its subtree.
//
// The minimum element in the tree is the root, at index 0.
//
// A heap is a common way to implement a priority queue. To build a priority
// queue, implement the Heap interface with the (negative) priority as the
// ordering for the Less method, so Push adds items while Pop removes the
// highest-priority item from the queue. The Examples include such an
// implementation; the file example_pq_test.go has the complete source.
//
package heap

import "sort"

// The Interface type describes the requirements
// for a type using the routines in this package.
// Any type that implements it may be used as a
// min-heap with the following invariants (established after
// Init has been called or if the data is empty or sorted):
//
//	!h.Less(j, i) for 0 <= i < h.Len() and 2*i+1 <= j <= 2*i+2 and j < h.Len()
//
// Note that Push and Pop in this interface are for package heap's
// implementation to call. To add and remove things from the heap,
// use heap.Push and heap.Pop.
type Interface interface {
  sort.Interface
  Push(x interface{}) // add x as element Len()
  Pop() interface{}   // remove and return element Len() - 1.
}

// Init establishes the heap invariants required by the other routines in this package.
// Init is idempotent with respect to the heap invariants
// and may be called whenever the heap invariants may have been invalidated.
// The complexity is O(n) where n = h.Len().
/* 对堆的单次操作是 nlogn ，但是这边采用的是下沉操作，因为数组的形式一开始可认为是个乱序的堆，所以这边从堆底往上操作，每次的操作对象是个高度为 i 的子堆，将子堆构造成一个符合条件的 大/小 堆后，有序向从局部到整体所以无需操作所有的节点
 */
func Init(h Interface) {
  // heapify
  n := h.Len()
  for i := n/2 - 1; i >= 0; i-- {
    down(h, i, n)
  }
}

// Push pushes the element x onto the heap.
// The complexity is O(log n) where n = h.Len().
func Push(h Interface, x interface{}) {
  // 数值加入堆尾
  h.Push(x)
  // 堆尾尝试上浮
  up(h, h.Len()-1)
}

// Pop removes and returns the minimum element (according to Less) from the heap.
// The complexity is O(log n) where n = h.Len().
// Pop is equivalent to Remove(h, 0).
func Pop(h Interface) interface{} {
  n := h.Len() - 1
  // 将堆顶数值和堆数组最后值做交换，而后操作这时候的堆顶（堆尾换上去的）进行下沉操作重构当前堆，最后弹出堆尾（最开始的堆顶，始终为之前堆结构的最值
  h.Swap(0, n) 
  // 下沉操作
  down(h, 0, n)
  return h.Pop()
}

// Remove removes and returns the element at index i from the heap.
// The complexity is O(log n) where n = h.Len().
func Remove(h Interface, i int) interface{} {
  n := h.Len() - 1
  if n != i {
    // 逻辑和 Pop api 类似，和堆尾交换
    h.Swap(i, n)
    if !down(h, i, n) {
      // 下沉失败则尝试上浮
      up(h, i)
    }
  }
  return h.Pop()
}

// Fix re-establishes the heap ordering after the element at index i has changed its value.
// Changing the value of the element at index i and then calling Fix is equivalent to,
// but less expensive than, calling Remove(h, i) followed by a Push of the new value.
// The complexity is O(log n) where n = h.Len().
func Fix(h Interface, i int) {
  if !down(h, i, h.Len()) {
    up(h, i)
  }
}
/* 堆元素上浮操作
 * 依次和父节点进行比较，如果比父节点 大/小（看堆性质）则与父节点进行交换
 */
func up(h Interface, j int) {
  for {
    i := (j - 1) / 2 // parent    根据笔记开头的下标对应反向计算获得父节点和当前 j 坐标的关系
    if i == j || !h.Less(j, i) {
      break
    }
    h.Swap(i, j)
    j = i
  }
}
/* 下沉操作
 * 依次和左右子节点比较，如果比子节点 大/小 （看堆性质）则进行交换
 */
func down(h Interface, i0, n int) bool {
  i := i0
  for {
    j1 := 2*i + 1
    if j1 >= n || j1 < 0 { // j1 < 0 after int overflow
      break
    }
    j := j1 // left child 先与左子节点进行比较
    if j2 := j1 + 1; j2 < n && h.Less(j2, j1) {
      j = j2 // = 2*i + 2  // right child 再与右子节点比较
    }
    if !h.Less(j, i) {
      break
    }
    // 如果子节点符合条件进行父子交换操作
    h.Swap(i, j)
    // 交换后的点即为新坐标，继续进行
    i = j
  }
  // 比较下沉后的坐标是否为原始坐标，以此判定是否下沉成功
  return i > i0
}

```

