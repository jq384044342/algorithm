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

