---
type: post  
title: ML Regression  
date: 2016/3/25  
tags: Note
categories: Machine Learning

---

## 1. ML: Regression
- Simple Regression
- Multiple Regression
- Assessing Performance  
- Ridge Regression
- Feature Selection & Lasso Regression 
- Nearest Neighbor & Kernel Regression 

### Models
- Linear Regression
- Regularization: Ridge(L2), Lasso(L1)
	- Nearest neighbor and kernel regression

### Algorithms
- Gradient descent
- Coordinate descent

### Concepts
- Loss functions 
- bias-variance tradeoff 
- cross-validation 
- sparsity  
- overfitting 
- model selection 
- feature selection

	
----
## Simple Regression
### Regression Fundamentals
- Data
	- input
	- output
- model
	- y_{i}=f(x_{i})+\varepsilon_{i}
	- \sum (\varepsilon_i) = 0 is expected 
	- it means y_{i} is eqully likely to be below or above f(x_{i})
- Task
	- which is model f(x)
	- for a given model f(x), estimate function f hat (x) from data 
- 
