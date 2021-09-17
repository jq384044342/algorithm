# Class 02 认知O(NlogN)的排序算法

**Master公式**

剖析递归行为和递归行为时间复杂度的估算

用递归方法找一个数组中的最大值，系统上到底是怎么运行的？
$$
T(N) = a*T(\frac{N}{b})+O(N^d)
$$

1. a代表次数，N/b代表子规模，O(N^d)代表其他常数操作

2. $$
   log_b^a < d,时间复杂度就是O(N^d)
   $$

3. $$
   log_b^a > d,时间复杂度就是O(N^(log_b^a))
   $$

4. $$
   log_b^a = d,时间复杂度就是O(N^d*logN)
   $$

5. 

```java
//求中点 mid = L + (R - L)>>1
public static int getMax(int[] arr){
    return process(arr,0,array.length-1)
}
public static int process(int[] arr,int L,int R){
    if(L==R){
        return;
    }
    int mid = L+(R-L)>>1;
    int leftMax = process(arr,L,mid);
    int rightMax = process(arr,mid+1,R);
    return Math.max(leftMax,rigthMax);
}
```

归并排序：

1.从中点将数据分为两个部分，左侧右侧分别排序，然后merge

2.merge的过程就是，左右各从最左比较，如果左边小于等于右边则把数据copy至新数组，否则就copy右侧，指针向下移动。如果有一方越界了，则copy剩余数据至目标数组。

```java
public static void mergeSort(int[] arr){
    if(null == arr||arr.length<2){
        return;
    }
    process(arr,0,arr.length-1);
}
public static void process(int[] arr,int L,int R){
    if(L==R){
        return;
    }
    int mid = L + ((R-L) >>1);
    process(arr,L,mid);
    process(arr,mid+1,R);
    merge(arr,L,mid,R);
}
public static void merge(int[] arr,int L,int M,int R){
    int[] help = new int[R-L+1];
    int i=0;
    int p1=L;
    int p2=M+1;
    while(p1<=M&&p2<=R){
        help[i++] = arr[p1]<=arr[p2]?arr[p1++]:arr[p2++];
    }
    while(p1<=M){
        help[i++]=arr[p1++];
    }
    while(p2<=R){
        help[i++]=arr[p2++];
    }
    for(i =0;i<help.length;i++){
        arr[L+i] = help[i];
    }
}
```

**Question:小和问题，左边的数比右边的数小计入小和，求一个数组的小和问题**

```java
//深度改写了merge的过程，右边有多少比左边大，这个数就记入小和多少次
public static int smallSum(int[] arr){
    if(null==arr||arr.length<2){
        return 0;
    }
    return process(arr,0,arr.length-1);
}
public static int process(int[] arr,int l,int r){
    if(l==r){
        return 0;
    }
    int mid = l+((r-l)>>1);
    return process(arr,l,mid)+process(arr,mid+1,r)+merge(arr,l,mid,r);
}
public static int merge(int[] arr,int l,int m,int r){
    int[] help = new int[L+R-1];
    int i=0;
    int p1=l;
    int p2=r;
    int res=0;
    while(p1<=m&&p2<=r){
        res +=arr[p1]<arr[p2]?(r-p2+1)*arr[p1]:0;
        help[i++]=arr[p1]<arr[p2]?arr[p1++]:arr[p2++];
    }
    while(p1<=m){
        help[i++]=arr[p1++];
    }
    while(p2<=r){
        help[i++]=arr[p2++];
    }
    for(int i=0;i<help.length;i++){
        arr[l+i]=help[i]
    }
    return res;
}
```

**Question:在一个数组中，如果左边的数比右边的数大，那么就构成逆序对，求一个数组有多少个逆序对？**

快速排序：

1.0 去最后一个数做基础，小于等于放左边，大于的放右边 O(N^2)

2.0 最后一个数作为目标数，小于的放左边，等于的放中间，大于的放右边，一次搞定一批数O(N^2)

因为划分值打的很偏，总能举出时间复杂度最糟糕的例子。所以需要随机选出目标数，进行partion过程。因为目标数是随机选出的，是概率时间，所以最终的时间复杂度O(N*logN),这就是快排3.0

3.0  每个partition随机选出目标数进行partition，额外的空间复杂度是O(logN)的

```java
public static void quickSort(int[] arr){
    if(null==arr||arr.length<2){
        return;
    }
    quickSort(arr,0,arr.length-1);
}
public static void quickSort(int[] arr,int L,int R){
    if(L<R){
        swap(arr,L+(int)(Math.random()*(R-L+1)),R);
        int[] p = partition(arr,L,R);
        quickSort(arr,L,p[0]-1);//<区
        quickSort(arr,L.p[1]+1,R);//>区
    }
}
//这是一个处理arr[l..r]的函数
//默认一arr[r]做划分，arr[r]->p  <p ==p >p
//返回等于区域的（左边界，右边界），所以返回一个长度为2的数组res,res[0] res[1]
public static int[] partition(int[] arr,int L,int R){
    int less = L-1;//<区右边界
    int more = R;//>区左边界
    while(L<more){//L表示当前数的位置 arr[R] ->划分值
        if(arr[L]<arr[R]){//当前数<划分值
            swap(arr,++less,L++);
        }else if(arr[L] > arr[R]){//当前数>划分值
            swap(arr,--more,L);
        }else{
            L++;
        }
    }
    swap(arr,more,R);
    return new int[](less+1,more);
    
}
```

**荷兰问题：给一个数组，一个数，将小于的数放左边，等于的放中间，大于的放右边**

```java
//从左到右遍历数组，当前数有三种情况：
//1.<num，和左边小于的右边界L交换，同时L++，i++ 
//2.==num,i++
//3.>num,和右边大于的左边界R交换，同时R--,i不变
public static void helanQuestion(int[] arr,int num){
        if(null == arr||arr.length<2){
            return;
        }
        int L = 0;
        int R = arr.length-1;
        for (int i = 0; i < arr.length && i!=R; i++) {
            if(arr[i]<num){
                swap(arr,L,i);
                L++;
            }else if (arr[i] >num){
                swap(arr,R,i);
                R--;
                i--;
            }
        }
    }
```

