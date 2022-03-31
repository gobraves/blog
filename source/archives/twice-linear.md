---
title: twice linear
date: 2019-01-14 00:20:54
categories: codewars
---
## Detail
Consider a sequence u where u is defined as follows:

The number u(0) = 1 is the first one in u.
For each x in u, then y = 2 * x + 1 and z = 3 * x + 1 must be in u too.
There are no other numbers in u.
Ex: u = [1, 3, 4, 7, 9, 10, 13, 15, 19, 21, 22, 27, ...]

1 gives 3 and 4, then 3 gives 7 and 10, 4 gives 9 and 13, then 7 gives 15 and 22 and so on...

Task:
Given parameter n the function dbl_linear (or dblLinear...) returns the element u(n) of the ordered (with <) sequence u.

Example:
dbl_linear(10) should return 22

Note:
Focus attention on efficiency


## Solution
```go
func DblLinear(n int) int {
    // your code
    list := make([]int, 0)
    s_cur := 0
    t_cur := 0
    list = append(list, 1)
    size := len(list)
    
    for size < n+1 {
      s_ele := list[s_cur] * 2 + 1
      t_ele := list[t_cur] * 3 + 1
      
      if s_ele < t_ele {
        list = append(list, s_ele)
        s_cur++
 
      } else {
        list = append(list, t_ele)
        t_cur++
        if s_ele == t_ele {
          s_cur++
        }
      }
      size++ 
    }
    return list[n]
      
    
}
```
