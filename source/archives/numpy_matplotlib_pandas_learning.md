title: numpy,matplotlib,pandas学习
date: 2016/4/5
description: numpy,matplitlib,pandas学习小结
tags: Note
---

## numpy,matplotlib,pandas三大工具学习
### numpy
首先它的主要对象是一个array. Numpy's array class is called ndarray.

attributes of ndarray object are:
- ndarray.ndim
- ndarray.shape
- ndarray.size
- ndarray.dtype
- ndarray.itemsize
- ndarray.data

#### Array creation
`a = np.array([[2,3,4],[2,3,4]])'  
`b = np.array([[1,2],[3,4]],dtype=complex)`  
`np.zeros((3,4))`  
`np.ones((2,3,4),dtype=np.int64)`  
#### create sequences of numbers
`np.arange(10,30,5)`  
`np.linspace(0,2,9)`
