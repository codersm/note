---
title: Java数据结构
date: 2017-01-28 10:57:47
tags: Java
---
# Java数据结构

## 2、简单排序算法

### 2.1、冒泡排序

冒泡排序（英语：Bubble Sort）是一种简单的排序算法。它重复地走访过要排序的数列，一次比较两个元素，如果他们的顺序错误就把他们交换过来。走访数列的工作是重复地进行直到没有再需要交换，也就是说该数列已经排序完成。这个算法的名字由来是因为越小的元素会经由交换慢慢“浮”到数列的顶端。

**冒泡排序算法的运作如下**

1. 比较相邻的元素。如果第一个比第二个大，就交换他们两个。
2. 对每一对相邻元素作同样的工作，从开始第一对到结尾的最后一对。这步做完后，最后的元素会是最大的数。
3. 针对所有的元素重复以上的步骤，除了最后一个。
4. 持续每次对越来越少的元素重复上面的步骤，直到没有任何一对数字需要比较。

**冒泡排序Java代码实现**

```java
// 冒泡排序
public void bubbleSort(int[] arr){

    int length = arr.length,out,in;

    for (out = length - 1; out > 1;out--){
        for(in = 0; in < out;in++){
            if(arr[in] > arr[in+1]){
                int temp = arr[in];
                arr[in] = arr[in+1];
                arr[in+1] = temp;
            }
        }
    }
}
```

### 2.2、选择排序

选择排序（Selection sort）是一种简单直观的排序算法。它的工作原理是每一次从待排序的数据元素中选出最小（或最大）的一个元素，存放在序列的起始位置，直到全部待排序的数据元素排完。 

<!-- 选择排序改进了冒泡排序，将必要的交换次数从O(N2)减少到O(N)。不幸的是比较次数保持为O(N2)。然而，选择排序依然为大记录量的排序提出了一个非常重要的改进，因为这些大量的记录需要在内存中移动，这就使交换的时间和比较的时间相比起来，交换的时间更为重要。 -->

**选择排序算法的运作如下**

数组中前一个元素跟后一个元素进行大小比较，如果后面的元素比前面的元素小则用一个变量k来记住他的位置，接着第二次比较，前面“后一个元素”现变成了“前一个元素”，继续跟它的“后一个元素”进行比较如果后面的元素比它要小则用变量k记住它在数组中的位置(下标)，等到循环结束的时候，我们应该找到了最小的那个数的下标了，然后进行判断，如果这个元素的下标不是第一个元素的下标，就让第一个元素跟他交换一下值，这样就找到整个数组中最小的数了。然后找到数组中第二小的数，让他跟数组中第二个元素交换一下值，以此类推。


**选择排序Java代码实现**

```java
// 选择排序
public void selectSort(int[] arr){

    int out,in,min,length = arr.length;

    for(out = 0; out < length - 1; out++){
        min = out;
        for(in = out + 1;in < length;in++){
            if (arr[in] < arr[min]){
                min = in;
            }
        }

        if(out != min){
            int temp = arr[out];
            arr[out] = arr[min];
            arr[min] = temp;
        }
    }
}
```

### 2.3、插入排序

插入排序（英语：Insertion Sort）是一种简单直观的排序算法。它的工作原理是通过构建有序序列，对于未排序数据，在已排序序列中从后向前扫描，找到相应位置并插入。

**插入排序算法的运作如下**

1. 认为第一个元素是排好序的，从第二个开始遍历。
2. 拿出当前元素(已排序)的值，从排好序的序列中从后往前找。
3. 如果序列中的元素比当前元素大，就把它后移。直到找到一个小的。
4. 把当前元素放在这个小的后面（后面的比当前大，它已经被后移了）。

**插入排序Java代码实现**

```java
// 插入排序
public void insertSort(int[] arr){

    int in, out,length = arr.length;

    for(out = 1; out < length; out++){

        int temp = arr[out];

        in = out;

        while (in > 0 && arr[in - 1] >= temp){
            arr[in] = arr[in-1];
            --in;
        }

        arr[in] = temp;
    }
}
```

在任意情况下，对于随机顺序的数据进行插入排序也需要O(N^2)的时间级；对于已经有序或基本有序的数据来说，插入排序要好得多，在这种情况下，算法运行只需要O(N)。

然而，对于逆序排序的数据，每次比较和移动都会执行，所以插入排序不必冒泡排序快。

## 4、递归

递归是一种方法（函数）调用自己的编程技术。递归不仅可以解决特定的问题，而且它也为解决很多问题提供了一个独特的概念上的框架。

三角数字 
1,3,6,10,15,21，...

```java
public static int triangle(int n){
    if(n == 1){
        return 1;
    }else {
        return (n + triangle(n-1));
    }
}
```

### 递归特征

1、调用自身。

2、当它调用自身的时候，是为了解决更小的问题。

3、存在某个足够简单的问题的层次，在这一层算法不需要调用自己就可以直接解答，且返回结果。

### 递归效率

调用一个方法会有一定的额外开销。控制必须从这个调用的位置转移到这个方法的开始处，此外，这个方法的参数以及方法返回的地址都要被压入到一个内部的栈里，为的是这个方法可以访问参数值和知道回到哪里。

另外一个低效性反映在系统内部空间存储所有的中间参数以及返回值，如果有大量的数据需要存储，就会引起栈溢出的问题。

### 归并排序

归并排序的思想是把一个数组分成两半，排序每一半，然后用merge()方法把数组的两半归并成一个有序的数组。

```java
public class MergeSortApp {

    public static void main(String[] args) {
        int maxSize = 100;
        DArray arr;
        arr = new DArray(maxSize);

        arr.insert(64);
        arr.insert(21);
        arr.insert(30);
        arr.insert(40);
        arr.insert(70);
        arr.display();

        arr.mergeSort();

        arr.display();
    }
}

class DArray{

    private long[] theArray;

    /**
     * number of data items
     */
    private int nElems;

    public DArray(int max){
        theArray = new long[max];
        nElems = 0;
    }


    public void insert(long value){
        theArray[nElems] = value;
        nElems++;
    }

    public void display(){
        for (int j = 0; j < nElems;j++){
            System.out.print(theArray[j]+" ");
        }
        System.out.println("");
    }

    public void mergeSort(){
        long[] workSpace = new long[nElems];
        recMergeSort(workSpace,0,nElems-1);
    }

    private void recMergeSort(long[] workSpace,int lowerBound,
                              int upperBound){
        if(lowerBound == upperBound){
            return;
        }else {

            int mid = (lowerBound+upperBound)/2;
            // sort low half
            recMergeSort(workSpace,lowerBound,mid);
            // sort high half
            recMergeSort(workSpace,mid+1,upperBound);
            // merge them
            merge(workSpace,lowerBound,mid+1,upperBound);
        }

    }

    private void merge(long[] workspace,int lowPtr,int highPtr,int upperBound){

        // workspace index
        int j = 0;
        int lowerBound = lowPtr;
        int mid = highPtr - 1;
        int n = upperBound - lowerBound + 1;

        while (lowPtr <= mid && highPtr <= upperBound){
            if(theArray[lowPtr] < theArray[highPtr]){
                workspace[j++] = theArray[lowPtr++];
            }else{
                workspace[j++] = theArray[highPtr++];
            }
        }

        while (lowPtr <= mid){
            workspace[j++] = theArray[lowPtr++];
        }

        while (highPtr <= upperBound){
            workspace[j++] = theArray[highPtr++];
        }

        for (j=0;j<n;j++){
            theArray[lowerBound++] = workspace[j];
        }
    }
}
```

> 归并排序的一个缺点是它需要在存储器中有另一个大小等于被排序的数据项目的数组。如果初始数组几乎占满整个存储器，那么归并排序将不能工作。但是，如果有足够的空间，归并排序会是一个很好的选择。


### 消除递归

在实际的运用中证明递归算法的效率不太高，在这种情况下，把递归的算法转换成非递归的算法是非常有用的。**这种转换经常会用到栈**。





## 高级排序

### 希尔排序

希尔排序基于插入排序，但是增加了一个新的特性，大大地提高了插入排序的执行效率。希尔排序对于多达几千个数据项的，中等大小规模的数组排序表现良好。希尔排序不像快速排序和其他时间复杂度为O(N*logN)的排序算法那么快，因此对非常大的文件排序，它不是最优选择。

希尔排序在最坏情况下的执行效率和平均情况下的执行效率相比没有差很多。一些专家提供差不多任何排序工作在开始时都可以使用希尔排序算法，若在实际中证明它不够快，再改换成诸如快速排序这样更高级的排序算法。

#### n-增量排序

希尔排序通过加大插入排序中元素之间的间隔，并在这些有间隔的元素中进行插入排序，从而使数据项能大跨度移动。当这些数据项排过一趟序后，希尔排序算法减少数据项的间隔再进行排序，依此进行下去。进行这些排序时数据项之间的间隔被称为增量，并且习惯用字母h来表示。

#### 减少间隔

## 划分

划分数据就是把数据分为两组，使所有关键字大于特定值的数据项在一组，使所有关键字小于特定值的数据项在另一组。

### 划分算法

划分算法由两个指针开始工作，两个指针分别指向数组的两头。

### 快速排序

快速排序算法本质上通过把一个数组划分为两个子数组，然后递归地调用自身为每一个子数组进行快速排序来实现的。但是，对这个基本的设计还需要进行一些加工。算法还必须要选择枢纽以及对小的划分区域进行排序。

### 基数排序


