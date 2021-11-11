# Class 03 详解桶排序以及排序内容大总结

#### **堆**

**1.一个完全二叉树。要么是满二叉树，要么是从左到右依次变满。**

**2.i位置的父节点下标(i-1)/2,左孩子2i+1，右孩子2i+2**

**3.大/小根堆：每一颗子树的最大值/最小值是头节点**

**4.堆结构的heapInsert与heapify操作**

**5.堆结构的增大和减少**

**6.优先级队列结构，就是堆结构**

```java
//与父比较，如果比父大就交换，直到不满足条件或到根节点
public static void heapInsert(int[] arr,index){
    while(arr[index]>arr[(index-1)/2]){
        swap(arr,index,(index-1)/2);
        index = (index-1)/2
    }
}
```



```java
//某个数在index位置，能否往下移动
public static void heapify(int[] arr,int index，int heapSize){
    int left = 2*i +1;
    while(left<heapSize){
        //两个孩子中，谁的值大，把下标给largeset
        int largest = left +1<heapSize&&arr[left+1]>arr[left]?left+1:left;
        //父和较大的孩子之间，谁的值大，把下标给largest
        largest = arr[largest]>arr[index]?largest:index;
        if(largest==index){
            break;
        }
        swap(arr,largest,index);
        index = largest;
        left = 2*index+1;
    }
}
```

 

```java
//堆排序
public static void heapSort(int[] arr){
    if(null==arr||arr.length<2){
        return;
    }
    for(int i=0;i<arr.length;i++){
        heapInsert(arr,i);
    }
    /*
    for(int i=arr.length-1;i>=0;i--){
    	heapify(arr,i,arr.length);
    }
    */
    int heapSize=arr.length;
    swap(arr,0.--heapSize);
    while(heapSize>0){
        heapify(arr,0,heapSize);
        swap(arr,0,--heapSize);
    }
}
```

**Question :已知一个几乎有序的数组，几乎是指，如果把数组拍好顺序的话，每个元素移动的距离可以不超过K，并且K相对于数组来说比较小。请选择一个合适的排序算法针对这个数据进行排序。**

```java
public static void kSort(int[] arr){
    //优先级队列就是默认就是小根堆
    PriorityQueue<Integer> heap = new PriorityQueue<>()
    int index = 0;
    for(;index<=Math.min(arr.length,k);index++){
        heap.add(arr[index]);
    }
    int i=0;
    for(;index<arr.length;i++,index++){
        heap.add(arr[index]);
        arr[i] = heap.poll();
    }
    while(!heap.isEmpty()>0){
        heap.poll();
    }
}
```

**比较器的使用**

1）比较器的实质就是重载比较运算符

2）比较器可以很好的应用在特殊标准的排序上

3）比较器可以很好的应用在根据特殊标准排序的结构上

**桶排序**

1.计数排序

准备一个范围词频表，遍历数据，并在相应的位置计数，然后按照顺序和频次吐出数据。

2.基数排序

以十进制为例，准备10个桶，按照低位向高位的顺序入桶出桶。因为数据的大小是高位优先，所以每次入桶出桶的顺序也保留下来了

```java
public static void radixSort(int[] arr){
    if(null == arr||arr.length<2){
        return;
    }
    radixSort(arr,0,arr.length-1,maxbits(arr));
}
public static int maxbits(int[] arr){
    int max = Integer.MIN_VALUE;
    for(int i=0;i<arr.length;i++){
        max = max <arr[i]?arr[i]:max;
    }
    int res = 0;
    while(max!=0){
        max = max/10;
    }
    return res;
}
public static void radixSort(int[] arr,int L,int R,int digit){
    final int radix =10;
    int i=0,j=0;
    int[] bucket = new int[R-L+1];
    for(int d =1;d<=digit;d++){//有多少位就进出多少次
        //10个空间
        //count[0]当前位(d位)是0的数字有多少个
        //count[1]当前位(d位)是0和1有多少个
        //count[2]当前位(d位)是0、1、2有多少个
        //count[i]当前位(d位)是0~i的数字有多少个
        int[] count=new int[radix];
        for(i=L;i<=R;i++){
            j = getDigit(arr[i],d);
            count[j]++;
        }
        for(i=1;i<radix;i++){
            count[i]=count[i]+count[i-1];
        }
        for(i=R;i>=L;i--){
            j=getDigit(arr[i],d);
            bucket[count[j]-1]=arr[i];
            count[j]--;
        }
        for(i=L,j=0;i<=R;i++,j++){
            arr[i] = bucket[j];
        }
    }
}
public static int getDigit(int x,int d){
    return ((x/(int)Math.pow(10,d-1))%10);
}
```

**时间复杂度、空间复杂度、和稳定性**

| 排序算法 | 时       | 空      | 稳   |
| -------- | -------- | ------- | ---- |
| 选择     | O(N^2)   | O(1)    | ❌    |
| 冒泡     | O(N^2)   | O(1)    | ✔    |
| 插入     | O(N^2)   | O(1)    | ✔    |
| 归并     | O(NlogN) | O(N)    | ✔    |
| 快排(随) | O(NlogN) | O(logN) | ❌    |
| 堆       | O(NlogN) | O(1)    | ❌    |

**常见的坑**

1. 归并排序的空间复杂度可以变成O(1),但是非常难，并且会破坏稳定性，归并排序内部缓存法

2. 原地归并排序的帖子都是垃圾，会让归并排序的时间复杂度变成O(N^2)
3. 快排可以做到稳定性，非常难，不需要掌握。可以搜01 satble sort
4. 所有的改进都不重要，因为目前没有找到时间复杂度O(NlogN)，额外空间复杂度O(1)，又稳定的排序
5. 有一道题目，是奇数放在数组左边，偶数放到数组右边，还要求原始的相对次序不变，碰到这个问题，可以怼面试官

**工程上的改进**

1. 充分利用O(N*logN)和O(N^2)排序各自的优势
2. 稳定性的考虑
3. 例如在快排或者归并排序，在样本量小于60的时候直接采用插入排序
4. Arrays.sort()类型是基础类型就是快排，对象类型是归并，是因为稳定性
