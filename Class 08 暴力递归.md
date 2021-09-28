# Class 08 暴力递归

**暴力递归就是尝试**

**1，把问题转化为规模缩小了的同类问题的子问题**

**2，有明确的不需要继续进行递归的条件(base case)**

**3，有当得到了子问题的结果之后的决策过程**

**4，不记录每一个子问题的解**

**一定要学会怎么去尝试，因为这是动态规划的基础，这一内容我们将在提升班讲述**

****

**汉诺塔问题**

**打印n层汉诺塔从最左边移动到最右边的全部过程**

```java
//from to other
//p(i,from,to,other)
//     出   目 另
//1)把1~i-1 从from移动到other去 2)把i从from移动到to去 3)把1~i-1从other移动到to
public static void hanoi(int n){
    if(n>0){
        func(n,"左","右","中")；
    }
}
public static void func(int i,String start,String end,String other){
    if(i==1){
        System.out.println("Move 1 from" +start +"to" +end);
    }else{
        func(i-1,start,other,end);
        System.out.println("Move" + i+"from"+start+"to" +end);
        func(i-1,other,end,start);
    }
}
```

**打印一个字符串的全部子序列，包括空串**

```java
public static void printAllSubsquence(String str){
    char[] chs = str.toCharArray();
    process(chs, 0, new ArrayList<Character>());
}
//当前来到i位置，要和不要，走两条路
//res是之前的选择，所形成的列表
public static void process(char[] str,int i, List<Character> res){
    if(i==chs.length){
        System.out.println(res);
        return;
    }
    List<Character> resKeep = copyList(res);
    resKeep.add(str[i]);
    process(str,i+1,reskeep);//要当前字符的路
    List<Character> resNoInclude = copyList(res);
    process(str, i+1, resNoInclude);//不要当前字符的路
}
```

**打印一个字符串的全部排列
打印一个字符串的全部排列，要求不要出现重复的排列**

```java
public static ArrayList<String> Permutation(String str){
    ArrayList<String> res = new ArrayList<>();
    if(str == null||str.length == 0){
        return res;
    }
    char[] chs = str.toCharArray();
    process(chs,0.res);
    return res;
}
//str[i..]范围上，所有的字符，都可以再i位置上，后续都去尝试
//str[0..i-1]范围上，是之前做的选择
//请把所有的字符串形成的全排列，加入到res里去
public static void process(char[] str, int i,ArrayList<String> res){
    if(i == str.length){
        res.add(String.valueof(str));
    }
    //boolean[] visit = new boolean[26];
    for(int j =i;j<str.length;j++){
        if(!visit[str[j]-'a']){
            visit[str[j]-'a'] =true;
            swap(str,i,j);
            process(str,i+1,res);
           //交换回去再交换回来，str保持不变
           swap(str,j,i);
        }
    }
    
}
public static void swap(char[] str,int i,int j){
    char tmp = str[i];
    str[i] = str[j];
    str[j] = tmp;
}
```

**给定一个整型数组arr，代表数值不同的纸牌排成一条线。玩家A和玩家B依次拿走每张纸牌，规定玩家A先拿，玩家B后拿，但是每个玩家每次只能拿走最左或最右的纸牌，玩家A和玩家B都绝顶聪明。请返回最后获胜者的分数。**

**[举例]**

**arr=[1,2,100,4]。**

**开始时，玩家A只能拿走1或4。如果开始时玩家A拿走1，则排列变为[2,100,4]，接下来玩家B可以拿走2或4，然后继续轮到玩家A.**

**如果开始时玩家A拿走4，则排列变为[1,2,100]，接下来玩家B可以拿走1或100，然后继续轮到玩家A..**

**玩家A作为绝顶聪明的人不会先拿4，因为拿4之后，玩家B将拿走100。所以玩家A会先拿1，让排列变为[2,100,4]，接下来玩家B不管怎么选，100都会被玩家A拿走。玩家A会获胜，分数为101。所以返回101。**

**arr=[1,100,2]。**

**开始时，玩家A不管拿1还是2，玩家B作为绝顶聪明的人，都会把100拿走。玩家B会获胜，**

**分数为100。所以返回100。**

```java
public static int win1(int[] arr){
    if(arr == null || arr.length ==0){
        return 0;
    }
    return Math.max(f(arr,0,arr.length-1),s(arr,0,arr.length-1));
}
public static int f(int[] arr,int i, int j){
    if(i==j){
        return arr[i];
    }
    return Math.max(arr[i]+s(arr,i+1,j),arr[j] +s(arr,i,j-1));
}

public static int s(int[] arr,int i,int j){
    if(i==j){
        return 0;
    }
    return Math.min(f(arr,i+1,j),f(arr,i,j-1));
}

public static int win2(int[] arr){
    if(arr==null||arr.length ==0){
        return 0;
    }
}
```

**给你一个栈，请你逆序这个栈，不能申请额外的数据结构，只能使用递归函数。如何实现?**

```java
public static void reverse(Stack<Integer> stack){
    if(stack.isEmpty()){
        return;
    }
    int i = f(stack);
    reverse(stack);
    stack.push(i);
}

public static int f(Stack<Integer> stack){
    int result = stack.pop();
    if(stack.isEmpty()){
        return result;
    }else{
        int last = f(stack);
        stack.push(result);
        return last;
    }
}
```

**规定1和A对应、2和B对应、3和C对应.**

**那么一个数字字符串比如111"，就可以转化为"AA"、"KA"和"AK"。**

**给定一个只有数字字符组成的字符串str，返回有多少种转化结果。**

```java
//i之前的位置，如何转化已经做过决定了
//i..有多少种转化的结果
public static int process(char[] str,int i){
    if(i==str.length){
        return 1;
    }
    if(str[i] =='0'){
        return 0;
    }
    if(str[i] == '1'){
        int res = process(str,i+1);//i自己作为单独的部分，后续有多少种方法
        if(i+1<str.length){
            res += process(str,i+2);//i和i+1作为单独的部分，后续有多多少种方法
        }
        return res;
    }
    if(str[i] == '2'){
        int res = process(str,i+1);//i自己作为单独的部分，后续有多少种方法
        //i和i+1作为单独的部分并且没有超过26，后续有多多少种方法
        if(i+1<str.length&& (str[i+1]>='0' && str[i+1] <= '6')){
            res += process(str,i+2)
        }
        return res;
    }
    return process(str,i+1);
}
```

**给定两个长度都为N的数组weights和values，weights[i]和values[i]分别代表i号物品的重量和价值。给定一个正数bag，表示一个载重bag的袋子，你装的物品不能超过这个重量。返回你能装下最多的价值是多少?**

```java
//i..的货物的自由选择，形成的而最大价值返回
public static int process1(int[] weights,int[] values
                          ,int i, int alreadyweight, int bag){
    if(alreadyweight > bag){
        return 0;
    }
    if(i == weights.length){
        return 0;
    }
    return Math.max(process1(weights,values,i+1,alreadyweight,bag)),
    values[i] + process1(weights,values,i+1,alreadyweight+weight[i],bag);
}


```

