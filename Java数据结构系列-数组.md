
# Java数据结构系列-数组

数组是应用最广泛的数据存储结构。它被植入到大部分编程语言中。由于数组十分易懂，所以它被用来作为介绍数据结构的起步点，并展示面向对象编程和数据结构之间的相互关系。

## 使用数组

### 创建数组

Java中两种数据类型：基本类型和对象类型。在许多编程语言中（如C++），数组也是基本类型，但在Java中把它们当作对象来对待，因此在创建数组时必须使用new操作符：

```java
 int[] intArray;
 intArray = new int[100];
```

[] 操作符对编译器来说是一个标志，它说明正在命名的是数组对象而不是普通的变量。还可以通过另一种语法来使用这个操作符，将它放在变量名的后面，而不是类型后面：

```java
int intArray[] = new int[100];
```

但是将[] 放在int后面会清楚地说明[]是数据类型的一部分，而不是变量名的一部分。

### 获取数组大小

数组有一个length字段，通过它可以得知当前数组大小（数据项的个数）：

```java
int arrayLength = intArray.length;
```

正如大多数编程语言一样，**一旦创建数组，数组大小便不可改变**。

### 访问数组数据项

数组数据项通过使用方括号中的下标数来访问。这与其他语言类似：

```java
temp = intArray[3];
intArray[7] = 66;
```

如果访问小于0或比数组大小大的数据项，程序会出现Array Index Out of Bounds（数组下标越界）的运行时差错误。数组的下标是从0开始的，也就是说第一个数据项的下标是0。

### 初始化

当创建数组之后，如果不另行指定，基本类型数组会自动赋值为0或0.0而对象数组会自动赋值为null对象。使用下面的语法可以对一个基本类型的数组初始化，赋入非空值：

```java
int[] intArray = {0,3,6,9,12,15};
```

### 数组使用的例子

```java
/**
 * HighArray对数组基本操作进行封装
 */
public class HighArray {

    private long [] a;

    private int nElems;

    public HighArray(int max){
        a = new long[max];
        nElems = 0;
    }

    /**
     * 查找
     */
    public boolean find(long searchKey){
        int j;
        for(j = 0; j < nElems;j++){
            if(a[j] == searchKey){
                break;
            }
        }
        if(j == nElems){
            return false;
        }
        else{
            return true;
        }
    }

    /**
     * 插入
     * @param value
     */
    public void insert(long value){
        a[nElems] = value;
        nElems++;
    }

    /**
     * 删除
     * @param value
     * @return
     */
    public boolean delete(long value){
       int j;
       for(j = 0; j < nElems;j++){
           if(value == a[j]){
               break;
           }
       }
       if(j == nElems){
           return false;
       }else{
           // 移动后面的元素
           for (int k = j; k < nElems;k++){
               a[k] = a[k+1];
           }
           nElems--;
           return true;
       }
    }

    /**
     * 遍历数组元素
     */
    public void display(){
        for(int j = 0; j < nElems; j++){
            System.out.print(a[j]+" ");
        }
        System.out.println();
    }
}
```

## 有序数组

假设一个数组，其中的数据项按关键字升序排列，即最小值在下标为0的单元上，每一个单元比前一个单元的值大。这种数组被称为有序数组。将数据按顺序排序的好处就是**可以通过二分查找显著地提高查找速度**。

当向这种数组中插入数据项时，需要为插入操作找到正确的位置：刚好在稍小值的后面，稍大值的前面。然后将所有比待插入的数据项大的值向后移以便腾出空间。

### 二分查找

二分查找使用的方法与我们在孩童时期常玩的猜数游戏中所用的方法一样。在这个游戏里，一个朋友会让你猜她正想的一个1至100之间的数。当你猜来一个数后，它会告诉你三种选择中的一个：你猜的比她想的大，或小，或猜中了。

为了能用最少的次数猜中，必须从50开始猜。如果她说你猜得太小，则推出那个数在51至100之间，所以下一次猜75。但如果她说有些大，则推出那个数在1至49之间，所以下一次猜25。

每猜一次就会将可能的值划分成两部分。最后范围会缩小到一个数字那么大，那就是答案。

### 有序数组的Java代码

```java
public class OrdArray {

    private long[] a;
    private int nElems;

    public OrdArray(int max){
        a = new long[max];
        nElems = 0;
    }

    public int size(){
        return nElems;
    }

    // 二分查找
    public int find(long searchKey){
        int lowerBound = 0;
        int upperBound = nElems - 1;
        int curIn;
        
        while (true){
            curIn = (lowerBound + upperBound) / 2;
            if(a[curIn] == searchKey){
                return curIn;
            }else if (lowerBound > upperBound){
                return nElems;   // can't find it
            }else {
                if(a[curIn] < searchKey){
                    lowerBound = curIn + 1;
                }else {
                    upperBound = curIn - 1;
                }
            }

        }
    }

    public void insert(long value){
        int j;
         // 找到待插入的位置
        for(j = 0; j < nElems;j++){
            if(a[j] > value){
                break;
            }
        }
        // 腾出空间
        for(int k = nElems;k > j;k--){ 
            a[k] = a[k-1];
        }
        a[j] = value;
        nElems++;
    }

    public boolean delete(long value){
        int j = find(value);
        if(j == nElems){
            return false;
        }else{
            for (int k = j;k < nElems;k++){
                a[k] = a[k+1];
            }
            nElems--;
            return true;
        }
    }

    public void display(){
        for(int j = 0; j < nElems; j++){
            System.out.print(a[j]+" ");
        }
        System.out.println();
    }
}
```

<!-- ### 大O表示法

汽车按尺寸被分为若干类：微型、小型、中型等等。在不提及具体尺寸的情况下，这些分类可以为我们所涉及到车的大小提供了
一个大致的概念。我们同样也需要一种快捷的方法来评价计算机算法的效率。在计算机科学中，这种粗略的度量方法被称作“大O”表示法。

- 无序数组的插入：常数
- 线性查找：与N成正比
- 二分查找：与log(N)成正比

**不要常数**

大O表示法同上面的公式比较类似，但它省去了常数K。当比较算法，并不在乎具体的微处理器芯片或编译器：真正需要比较的是对应不同的N值，T是如何变化的，而不是具体的数字。因此不需要常数。 -->

