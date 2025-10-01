# 排序算法 - 快速排序

作为排序之王的快速排序法，要理解它很容易，但实现起来又是另外一回事，主要是因为快速排序在实现上是很苛刻的，差之毫厘，谬以千里，如果不是专业研究算法的话，突然哪天要用到还真不一定写的出来，主要是平时用的很少。。 

## 基本原理

  1. 给定一个数据集，先从中选出一个元素作为枢钮元，选取枢钮元有多种策略，它对算法的性能影响颇大，目前效果最好的应该是三数中值分割法，三数指的是数组中第一个元素，最后一个元素，和中间那个元素，然后从这三个元素中找出中位数并作为枢钮元
  2. 将大于枢钮元的元素移动到枢钮元的左边，将小于枢钮元的元素移动到枢钮元的右边
  3. 这时在枢钮元两边可以看作是两个子数据集，然后对这两个数据集分别进行快速排序，直到子数据集长度为1

## 时间复杂度

* 平均时间为O(N log N)
* 最坏情况为O(N^2) 

## 实例
```
static void swap(int [] a,int i,int j){
    int temp = a[i];
    a[i] = a[j];
    a[j] = temp;
}
static int partition(int a[],int start,int end){
    int m = start+(end-start)/2;
    if(a[start] > a[end])
        swap(a,start,end);
    if(a[start] > a[m])
        swap(a,start,m);
    if(a[m] < a[end])
        swap(a,m,end);
 
    return a[end];
}
static void quickSort(int a[],int start,int end){
 
    if(!(start<end))
        return;
 
    int temp = partition(a,start,end);
 
    int i = start, j = end;
    while(i < j){
        while(i < j && a[i] <= temp)i++;
        a[j] = a[i];
        while(i < j && a[j] >= temp)j--;
        a[i] = a[j];
    }
    a[j] = temp;
 
    quickSort(a,start,j-1);
    quickSort(a,j+1,end);
 
}
 
public static void quickSort(int[] arr){
    quickSort(arr, 0, arr.length - 1);
}
```
关于
---

__作者__：张佳军

__阅读__：17

__点赞__：3

__创建__：2017-06-24
