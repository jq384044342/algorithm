# Class 01认识复杂度和简单排序算法

#### 时间复杂度：最坏情况下，算法所执行的常数操作表达式，忽略低阶向和常数向，只剩高阶项并且去掉高阶项系数，用big O表示。

**选择排序：遍历数组，依次找出最小值，放到前面**

```java
public static void selectSort(int[] array){
    if(null == array||array.length<2)
        return;
    for(int i = 0;i< array.length-1;i++){
        int min = i;
        for(int j = i+1;j<array.length;j++){
            if(array[j]<array[min]){
                min = j;
            }
        }
        swap(array,i,min);
    }
}
public static void swap(int[] array,int a,int b){
    int temp = array[a];
    int a = array[b];
    int b = array[a];
}
```

**冒泡排序：遍历数组，每次比较相邻两个数，大的数往后移**

```java
public static void popSort(int[] array){
    if(null == array||array.length < 2)
        return;
    for(int i = 0;i<array.length-1;i++){
        for(int j = 0;j<array.length -i;j++){
            if(array[j]>array[j+1]){
                swap(array,j,j+1);
            }
        }
    }
}
```

***异或运算：二进制位运算，相同为0，不同为1。也可以理解为无进位相加。***

**0^N=N,N^N=0;满足交换律和结合律：a^b=b^a  a^b^c =a^c^b =b^c^a**

**所以两数交换还可以这么写**(*只有两数是独立的两块内存区域，否则将抹成0*)

```java
public static void swap(int a, int b){
    a = a ^ b;
    b = a ^ b;
    a = a ^ b;
}
```

**Question1:某组数中只有一种数出现了奇数次，用O(N)的算法实现**

```java
public static int find(int[] array){
    int eor = 0;
    for(int num : array){
        eor = eor ^ num;
    }
    return eor;
}
```

**Question2:某组数中只有两种数出现了奇数次，用O(N)的算法实现**

```java
public static void find(int[] array){
    int eor = 0;
    for(int num : array){
        eor = eor ^num;
    }
    //eor = a^b
    //rigthone：因为a与b不同，所以a与b一定在某一位上存在一个是1，一个是0。下面的表达式就是将最右侧的1提取出来。
    int rigthone = eor & (^eor+1);
    int eor1 = 0;
    for(int num : array){
        if(0 == num ^ rigthone){
            eor1 = eor1 ^ num;
        }
    }
    int a = eor1;
    int b = eor^eor1;
}
```

**插入排序：遍历数组，假设前方数组有序，将下一个数字从后向前比较，如果小于就前移，知道不满足为止。**

```java
public static void insertSort(int[] array){
    if(null == array||array.length<2)
        return;
    for(int i=0;i<array.length-1;i++){
        for(int j=i+1;j>0 && array[j]<array[j-1];j--){
            swap(array,j,j-1);
        }
    }
}
```

**二分法**

1. **在一个有序的数组中，找出某个数是否存在**
2. **在一个有序数组中，找出大于等于某个数的最左位置**
3. **局部最小值问题：数组无序，但是任何两个相邻的数不相等，求一个局部最小的问题**

**对数器**：

1. **你有一个想测的方法**a
2. **实现复杂度不好但是容易实现或者系统提供的方法**b
3. **实现一个随机样本产生器**
4. **把方法a和方法b跑相同的随机样本，看看得到的结果是否一致**
5. **如果有一个随机样本的对比结果不一致，打印样本进行人工干预，改对方法a或者b**
6. **当样本数量很多对比测试仍然正确，可以确定方法a已经正确**  

```java
//Math.random()产生出[0,1)所有的小数等概率返回
//N*Math.random()产生出[0,N)所有的小数等概率返回
//(int)N*Math.random()产生出[0,N-1]所有的整数等概率返回
public static int[] generateRandomArray(int maxsize,int maxvalue){
    int[] array = new int[((int)(maxsize+1)*Math.random())];
    for(int i=0;i<array.length-1;i++){
        array[i] = (int)(maxvalue+1)*Math.random()-(int)maxvalue*Math.random();
    }
}
public static void main(String[] args){
    int testTime = 500000;
    int maxsize = 100;
    int maxvalue = 100;
    for(int i=0;i<testTime;i++){
        int[] array = generateRandomArray(maxsize,maxvalue);
        int[] array2 = copyAray(array);
        sortA(array);
        sortB(array2);
        if(!isEqual(array,array2)){
            succeed = false;
            break;
        }
    }
}
```

