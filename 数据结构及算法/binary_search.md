### binary search

```go
/* file go/src/sort/search.go
 * 二分数组长度
 * 二分函数
 */
func Search(n int, f func(int) bool) int {
  // Define f(-1) == false and f(n) == true.
  // Invariant: f(i-1) == false, f(j) == true.
  i, j := 0, n
  for i < j {
    // 避免整形相加过程中溢出导致计算出错
    h := int(uint(i+j) >> 1) // avoid overflow when computing h
    // i ≤ h < j
    if !f(h) {
      i = h + 1 // preserves f(i-1) == false
    } else {
      j = h // preserves f(j) == true
    }
  }
  // i == j, f(i-1) == false, and f(j) (= f(i)) == true  =>  answer is i.
  return i
}
```

