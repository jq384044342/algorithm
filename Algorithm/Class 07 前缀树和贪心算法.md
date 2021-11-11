# Class 07 前缀树和贪心算法

**何为前缀树？如何生成前缀树**

例子：一个字符串类型的数组arr1，另一个字符串类型的数组arr2。arr2中有哪些字符，是arr1中出现的?请打印。arr2中有哪些字符，是作为arr1中某个字符串前缀出现的﹖请打印。arr2中有哪些字符，是作为arr1中某个字符串前缀出现的?请打印 arr2中出现次数最大的前缀。

```java
public class Code01_TrieTree{
    
    public static class TrieNode{
        public int pass;
        public int end;
        public TrieNode[] nexts;//HashMap<Char,Node> nexts;
        
        public TrieNode(){
            pass = 0;
            end = 0;
            //nexts[0] == null 没有走向a的路
            //nexts[0] != null 有走向a的路
            //..
            //nexts[25] !=null 有走向z的路
            nexts = new TrieNode][26];
        }
    }
    
    public static class Trie{
        private TrieNode root;
        
        public Trie(){
            this.root = new TrieNode();
        }
        
        public void insert(String word){
            if(word == null){
                return;
            }
            char[] chs = word.toCharArray();
            TrieNode node = root;
            node.pass++;
            int index = 0;
            for(int i = 0;i< chs.length;i++){//从左往右遍历字符
                index = chs[i]-'a';//由字符，对应成走向哪条路
                if(node.nexts[index] == null){
                    node.nexts[index] = new TrieNode();
                }
                node = node.nexts[index];
                node.pass++;
            }
            node.end++;
        }
        
        public void delete(String word){
            if(search(word) != 0){//确定树种确实加入过word，才删除
                char[] chs = word.toCharArray();
                TrieNode node = root;
                node.pass--;
                int index = 0;
                for(int i=0;i<chs.length;i++){
                    index = chs[i] -'a';
                    if(--node.nexts[index].pass == 0){
                        node.nexts[index] = null;
                        return;
                    }
                    node = node.nexts[index];
                }
                node.end--;
            }
            
        }
        
        //word这个单词之前加入过几次
        public int search(String word){
            if(word == null){
                return 0;
            }
            char[] chs = word.toCharArray();
            TrieNode node = root;
            int index = 0;
            for(int i = 0;i< chs.length;i++){
                index = chs[i] -'a';
                if(node.nexts[index] == null){
                    return 0;
                }
                node = node.nexts[index];
            }
            return node.end;
        }
        
       public int prefixNumber(String pre){
           
           if(word == null){
                return 0;
            }
           
            char[] chs = word.toCharArray();
            TrieNode node = root;
            int index = 0;
            for(int i = 0;i< chs.length;i++){
                index = chs[i] -'a';
                if(node.nexts[index] == null){
                    return 0;
                }
                node = node.nexts[index];
            }
            return node.pass;
       } 
    }
    
}
```

**贪心算法**

**再有一个标准下，优先考虑最满足标准的样本，最后考虑最不满足标准的样本，最终得到一个答案的算法，叫做贪心算法。**

**也就是说，不从整体最有上加以考虑，所作出的是在某种意义上的局部最优解**

**局部最优--？-->整体最优**

****

**贪心算法的在笔试时的解题套路**

**1，实现一个不依靠贪心策略的解法X，可以用最暴力的尝试**

**2，脑补出贪心策略A、贪心策略B、贪心策略C.**

**3，用解法X和对数器，去验证每一个贪心策略，用实验的方式得知哪个贪心策略正确4，不要去纠结贪心策略的证明**

****

**问题：一些项目要占用一个会议室宣讲，会议室不能同时容纳两个项目的宣讲。给你每一个项目开始的时间和结束的时间(给你一个数组，里面是一个个具体的项目)，你来安排宣讲的日程，要求会议室进行的宣讲的场次最多。**

```java
//哪个会议结束时间早，就先安排哪个会议
public class Code04_BestArrange{
    
    public static class Program{
        public int start;
        public int end;
        
        public Program(int start,int end){
            this.start = start;
            this.end = end;
        }
    }
    
    public static class ProgramComparator implements Comparator<Program>{
        
        @Override
        public int compare(Program o1,Program o2){
            return o1.end - o2.end;
        }
    }
    
    public static int bestArrange(Program[] programs,int timePoint){
        Arrays.sort(programs,new ProgramComparator());
        int result = 0;
        for(int i = 0;i < programs.length; i++){
          if(timePoint <= programs[i].start){
              result++;
              timepoint = programs[i].end;
          }  
        }
        return result;
    }
    
}
```

**给一个字符串数组，将他们拼接起来，字典序最小**

```java
public class Code02_LowestLexicography{
    
    public static class MyComparator implements Comparator<String>{
        @Override
        public int compare(String a,String b){
            return (a+b).compareTo(b+a);
        }
    } 
    
    public static String lowestString(String[] strs){
        if(strs == null ||strs.length==0){
            return "";
        }
        Arrays.sort(strs,new MyComparator());
        String res = "";
        for(int i=0;i<strs.length;i++){
            res += strs[i];
        }
        return res;
    }
    
}
```

**贪心策略在实现时，经常使用到的技巧:**

1. **根据某标准建立一个比较器来排序**
2. **根据某标准建立一个比较器来组成堆**

****

**Question：一块金条切成两半，是需要花费和长度数值一样的铜板的。比如长度为20的金条，不管切成长度多大的两半，都要花费20个铜板。**

**一群人想整分整块金条，怎么分最省铜板?**

**例如,给定数组{10,20,30}，代表一共三个人，整块金条长度为10+20+30-60。金条要分成10,20,30三个部分。如果先把长度60的金条分成10和50，花费60;再把长度50的金条分成20和30，花费50;一共花费110铜板。**

**但是如果先把长度60的金条分成30和30，花费60;再把长度30金条分成10和20，花费30;一共花费90铜板。**

**输入一个数组，返回分割的最小代价。**

```java
//哈夫曼编码问题
public class Code03_LessMoneySplitGold{
    
    public static int lessMoney(int[] arr){
        PriorityQueue<Integer> pQ = new PriorityQueue<>();
        for(int i=0;i < arr.length;i++){
            pQ.add(arr[i]);
        }
        int sum = 0;
        int cur = 0;
        while(pQ.size()>1){
            cur = pQ.poll()+pQ.poll();
            sum += cur;
            pQ.add(cur);
        }
        return sum;
    }
    
}
```

**Question:输入:**

**正数数组costs**

**正数数组profits**

**正数k,**

**正数m**

**含义:**

**costs[i]表示i号项目的花费**

**profits[i]表示i号项目在扣除花费之后还能挣到的钱(利润)**

**k表示你只能串行的最多做k个项目**

**m表示你初始的资金**

**说明:**

**你每做完一个项目，马上获得的收益，可以支持你去做下一个项目。**

**输出:**

**你最后获得的最大钱数。**

```java
public class Code05_IPO{
    
    public static class Node{
        public int p;
        public int c;
        
        public Node(int p, int c){
            this.p = p;
            this.c = c;
        }
    }
    
    public static class MinCostComparator implements Comparator<Node>{
        
        @Override
        public int compare(Node o1, Node o2){
            return o1.c-o2.c;
        }
    }
    
    public static class MaxProfitComparator implements Comparator<Node> {
        
        
        @Override
        public int compare(Node o1, Node o2){
            return o2.p - o1.p;
        }
    }
    
    publc static int findMaximizedCapital(int k,int w,int[] Profits,int[] Capital){
        
        PriorityQueue<Node> minCostQ = new PriorityQueue<>(new MinCostComparator());
        PriorityQueue<Node> maxProfitQ = new PriorityQueue<>(new MaxProfitComparator());
        //所有项目扔到锁池中，花费组织的小根堆
        for(int i=0;i<Profits.lengt;i++){
            minCostQ.add(new Node(Profits[i],Capital[i]));
        }
        for(int i=0;i<k;i++){//进行k轮
            //能力所及的项目，全解锁
            while(!minCostQ,isEmpty()&&minCostQ.peek().c<=k){
                maxProfitQ.add(minCostQ.poll());
            }
            if(maxProfitQ.isEmpty()){
                return W;
            }
            W += maxProfitQ.poll().p;
        }
        return W;
        
    }
}
```

**一个数据流中，随时可以取得中位数**

```java
//维持2个堆，一个大根堆，一个小根堆
//第一个数入大根堆
//每来一个数，判断是否比大根堆堆顶大，大就放入大根堆，否则就放入小根堆
//当大小根堆之间的size差=2时，一个堆堆顶弹出入另一个堆
```

**N皇后问题**

**N皇后问题是指在N*N的棋盘上要摆N个皇后，要求任何两个皇后不同行、不同列，也不在同一条斜线上。给定一个整数n，返回n皇后的摆法有多少种。**

**n=1，返回1。**

**n=8，返回92。**

**n=2或3，2皇后和3皇后问题无论怎么摆都不行，返回0。**

```java
public class Code09_NQueens {
    public static int num1(int n){
        if(n < 1){
            return 0;
        }
        int[] record = new int[n];//record[i] ->i行的皇后，放在了第几列
        return process1(0,record,n);
    }
    
    public static int process1(int i,int[] record, int n){
        if(i==n){//终止行
            return 1;
        }
        int res = 0;
        for(int j=0;j<n;j++){//当前在第i行，尝试i行所有的列->j
            //当前i行的皇后，放在j列，会不会和之前（0...i-1）的皇后，共行共列或者共斜线
            //如果是，认为无效
            //如果不是，认为有效
            if(isValid(record,i,j)){
                record[i] = j;
                res +=process1(i+1,record,n);
            }
            
        }
        return res;
    }
    
    //record[0...i-1]你需要看，record[i...]不需要看
    //返回i行皇后，放在了j列，是否有效
    public boolean isValid(int[] record,int i,int j){
        for(int k =0;k<i;k++){//之前的某个k行的皇后
            if(j==record[k]||Math.abs(record[k]-j)==Math.abs(i-k)){
                return false;
            }
        }
        return true;
    }
    
    public static int num2(int n){
        if(n<1||n>32){
            return 0;
        }
        int limit = n==32?-1:(1<<n)-1;
        return process2(limit,0,0,0);
    }
    
    
    //colLim 列的限制，1的位置不能放皇后，0的位置可以
    //leftDiaLim左斜线的限制，1的位置不能放皇后，0的位置可以
    //rightDiaLim右斜线的限制，1的位置不能放皇后，0的位置可以
    public static int process2(int limit,int colLim,int leftDiaLim,int rightDiaLim){
        if(colLim==limit){
            return 1;
        }
        int pos = 0;
        int mostRightOne = 0;
        //所有候选放皇后的位置，都在pos上
        pos = limit & (~(colLim|leftDiaLim|rightDiaLim));
        int res =0;
        while(pos!=0){
            mostRightOne = pos & (~pos+1);
            pos = pos - mostRightOne;
            res += process2(limit,colLim|mostRightOne,(leftDiaLim|mostRightOne)<<1,(rightDiaLim |mostRightOne)>>>1)
        }
    }
}
```

