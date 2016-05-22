---
layout: post
title: 排序整理
date: 2015-06-22
tags: 算法
categories: Note
---


>  《算法》第四版复习整理（c语言）

	void swap(int &i, int &j)
	{
		int temp = i;
		i = j;
		j = temp;
	}    

## 冒泡排序   
	void BubbleSort(int a[], int n)
	{
		for(int i = 0; i < n; i++)  
			for (int j = 1; j < n - i; ++j)  
			{
				if(a[j-1] > a[j])
				{
					swap(a[j-1],a[j]);
				}
			}
	}


## 选择排序
```
	void SelectSort(int a[], int n)
	{
		int i,j,min;
		int temp;
		for (i = 0; i < n; ++i)
		{
			min = i;
			for (j = i+1; j < n; ++j)
			{
				if (a[min] > a[j])
				min = j;
			}
			swap(a[i],a[min]);
		}
	}
```

## 插入排序   
```
	void InsertSort(int a[], int n)
	{
		int i,j;
		for (int i = 0; i < n; ++i)
		{
			for (int j = i; j > 0 && (a[j] < a[j-1]) ; --j)
			swap(a[j],a[j-1]);
		}
	}
```
## 希尔排序   
```
	void ShellSort(int a[], int n)
	{
		int h = 1;
		int i,j;
		while(h < n/3)
		h = 3*h + 1;
		while(h >= 1)
		{
			for (i = h; i < n; ++i)
			{
				for (j = i; j >= h && (a[j] < a[j-h]); j -= h)
				swap(a[j],a[j-h]);
			}
			h = h/3;
		}
	}
```


## 快速排序

### 1 edition   
```
		void QuickSort(int v[], int left, int right)
		{
			int i, last;
			if (left >= right)
				return;
			swap(v[left], v[(left+right)/2]);
				last = left;
			for (i = 0; i < right; ++i)
				if (v[i] < v[left])
					swap(++last, i);
			swap(left,last);
			QuickSort(v,left,last-1);
			QuickSort(v,last+1,right);
		}  
```
### 2 edition  
```
    	void QuickSort(int a[], int n)
    	{
    		//1.random shuffle 消除对输入的依赖
    		//2.Sort(a,0,n-1);
   	 	}

    	void Sort(int a[], int lo, int hi)
    	{
    		if (hi > lo)
    		{
    			int j = partition(a, lo, hi);
    			sort(a, lo, j-1);
    			sort(a, j+1, hi);
    		}
    	}

    	int partition(int a[], int lo, int hi)
    	{
       		// hi + 1 是一个技巧
       		int i = lo, j = hi+1;
       		int v = a[lo];
       		while(true)
       		{
       		while (a[++i] < v)
       		if (i == hi)
       		break;
       		while (v < a[--j])
       		if (j == lo)
       		break;
       		if (i >= j)
       		break;
       		swap(a[i],a[j]);
        	}
       		swap(a[lo],a[j]);
       		return j;
    	}
```
### 3 edition  
```
    	void QuickSort(int a[], int lo, int hi)
    	{
    		int i = lo, j = hi;
    		int temp;
    		if (hi > lo)
    		{
    			temp = a[lo];
    			while (i != j)
    			{
    				while (j > i && a[j] >= temp)
    				{
    				j--;
    				}
    				a[i] = a[j];
    				while (i < j && a[i] <= temp)
    				{
    					i++;
    				}
    				a[j] = a[i];
				}
				a[i] = temp;
				QuickSort(a, lo , i - 1);
				QuickSort(a, i + 1, hi);
    		}
    	}
```
> 版本2和版本3的区别在于，版本2的是让两边同时进行比较，在两边都不符合条件的情况下交换，直至i和j相等。版本3若是右边一边比较不符合条件，则移至另一边比较，直至i和j相等。

## 归并排序
```
    bool MergeSort(int a[], int n);
    {
       int aux[n];
 	   sort(a,0,n-1,aux);
 	   return true;
    }

    //自底向下
    void Sort(int a[], int lo, int hi,int aux[])
    {
       if (hi > lo)
       {
       	  int mid = lo + (hi - lo)/2;
       	  Sort(a, lo, mid);
       	  Sort(a, mid+1, hi);
       	  Merge(a, lo, mid, hi, aux);
       }
    }

    //自底向上
    void Sort(int a[], int n, int aux)
    {
       for(int sz = 1; sz < n; sz = sz + sz)
          for(int lo = 0; lo < n - sz; lo += sz+sz)
            // c语言没有min函数
             Merge(a, lo, lo+sz-1; Math.min(lo+sz+sz-1, n-1));
    }

	void Merge (int a[], int lo, int mid, int hi, int aux[])
	{
		int i = lo, j = mid + 1;
		//辅助变量
		int k;
		for(k = lo; k <= hi; k++)
		{
			aux[k] = a[k];
		}

		for (k = lo; k <= hi; k++)
			if      (i > mid)           a[k] = aux[j++];
			else if (j > hi )           a[k] = aux[i++];
			else if (aux[j] < aux[i])   a[k] = aux[j++];
			else                        a[k] = aux[i++];
	}
```
