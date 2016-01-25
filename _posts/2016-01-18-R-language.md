---
layout: post
title: R language
date: 2016-01-18
tags: R ml
excerpt:
---


##R language
###atomic class of object

- character
- numeric(double precision real number)
- integer(suffix L example:1L. It's explicitly an integer)
- complex
- logical

###basic object : vector
> vector can only contain objects of the same class

- Inf(represent infinity example: 1/0)
- NaN(not a number. eg: 0/0)


###Attributes
R objects can have attributes
- name,dimnames
- dimensions(e.g. matrices, arrays)
- class
- length
- other user-defined attributes/metadata
attributes of an objec can be accessed using the attributes().

###Coercion

####Implicit

Example:
- `x <- c("TRUE","a",123)` --character  
- `x<-c(TRUE,1,2,3)` --numeric  
- `x<-c(TRUE,1L,2L,3L)` --integer  

####explicit
Example: `x <- 0:6`  
- `as.numeric(x)`  
- `as.logical(x)`  
- `as.character(x)`  

###list
a special type of vector that can contain elements of different classes

###matrices  
every element must be in the same class  
`matrix(nrow = , ncol =  )`   
`dim()`  
`attributes()`  
- upper left  
  e.g.
  - `m <- matrix(1:6, nrow = 2, ncol = 3)`  
  - `m <- 1:6`  
    `dim(m) <- c(2,3)`  
- cbind() and rbind()  

###Factors   
`x <- factor(c("yes","yes","no","yes"))`  

###Missing Values  
- `is.na()` NA has class  
- `is.nan()` NAN is also a NA

###data frame  
can have different classes of objects in each column  
special attributes:
- row.names  
- created by calling `read.csv()` or `read.table()`  
- converted to a matrix by calling data.matrix()  
