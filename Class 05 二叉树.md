# Class 05 二叉树

**二叉树节点结构**

```java
class Node<V>{
    V value;
    Node left;
    Node right;
}
```

**用递归和非递归两种方式实现二叉树的先序、中序、后序遍历**

**如何直观的打印一颗二叉树**

**如何完成二叉树的宽度优先遍历（常见题目：求一颗二叉树的宽度）**

```java
//递归序，每一个节点都会来到三次，第一次就打印是先序，第二次打印是中序，第三次打印是后续
//先序，每一颗子树来说都是先头左右
public static void first(Node head){
    if(head==null){
        return;
    }
    System.out.println(head.value);
    first(head.left);
    first(head.right);
}
//中序，每一颗子树来说都是先左头右
public static void mid(Node head){
    if(head==null){
        return;
    }    
    mid(head.left);
    System.out.println(head.value);
    mid(head.right);
}
//后序，每一颗子树来说都是先左右头
public static void last(Node head){
    if(head==null){
        return;
    }
    last(head.left);
    last(head.right);
    System.out.println(head.value);
}
```

```java
//任何递归都可以改写成非递归
//先序，准备一个栈，1从栈中弹出一个节点cur 2打印cur 3先右后左压入栈中 重复以上三个步骤
public static void first(Node head){
    if(null == head){
        return;
    }
    if(head != null){
        Stack<Node> stack = new Stack<>();
        stack.add(head);
        while(!stack.isEmpty()){
            head = stack.pop();
            System.out.println(head.value);
            if(head.right!=null){
                stack.push(head.right);
            }
            if(head.left!=null){
                stack.push(head.left);
            }
        }
    }
}
//后序
//准备2个栈，第一个栈按照头右左的顺序入栈。1从栈中弹出一个节点cur 2.压入第二个栈  2先左后右压入栈1
public static void last(Node head){
    if(null==head){
        return;
    }
    if(head !=null){
        Stack<node> s1 = new Stack<>();
        Stack<node> s2 = new Stack<>();
        stack1.push(head);
        while(cur != null){
            head = s1.pop();
            s2.push(head);
            if(head.left!=null){
                s1.push(head.left);
            }
            if(head.right!=null){
                s1.push(head.right);
            }
        }
        while(!s2.isEmpty()){
            System.out.println(s2.pop().value);
        }
    }
}
//中序 
//每颗子树，整棵树左边界进栈,依次弹的过程中，对弹出节点的右树左边界入栈周而复始
pusblic static void mid(Node head){
    if(head!=null){
        Stack<Node> stack = new Stack<>();
        while(!stack.isEmpty()||head!=null){
            if(head!=null){
                stack.push(head);
                head = head.left;
            }else {
                head = stack.pop();
                System.out.println(head.value);
                head=head.right;
            }
        }
    }
}
```

**如何完成二叉树的宽度优先遍历（常见题目：求一颗二叉树的宽度）**

```java
//队列，1.头放入队列，打印 2.先左后右入队列
//求一颗二叉树的最大宽度
//要求WFS的时候需要在哪一层，并统计节点个数
public static void WFS(Node head){
    if(head==null){
        return;
    }
    Queue<Node> queue = new LinkedList<>();
    queue.add(head);
    HashMap<Node,Inetger> levelMap = new HashMap<>();
    levelMap.put(head,1);
    int curLevel =1;
    int curLevelNodes=0;
    int max = Integer.MIN_VALUE;
    while(!queue.isEmpty()){
        Node cur = queue.poll();
        int curNodeLevel = levelMap.get(cur);
        if(curNodeLevel== curLevel){
            curLevelNodes++;
        }else{
            max = Math.max(max,curLevelNodes);
            curLevel++;
            curLevelNodes =1;
        }
        Syetem.out.println(cur.value);
        if(cur.left!=null){
            levelMap.put(cur.left,curNodeLevel+1);
            queue.add(cur.left);
        }
        if(cur.right!=null){
            levelMap.put(cur.right,curNodeLevel+1);
            queue.add(cur.right);
        }
    }
}
```

**二叉树的相关概念及其实现判断**

**如何判断一棵二叉树是否是搜索二叉树(BST)**

```java
//每一颗子树的左数节点都比他小，右数都比它大
//中序遍历是升序既是搜索二叉树
public static int preValus = Ineteger.MIN_VALUE;
public static void isBST(Node head){    
    if(head==null){
        return true;
    }
    boolean isLeftBST=isBST(head.left);
    if(!isLeftBST){
        return false;
    }
    if(head.value<=preValue){
        return false;
    }else{
        preValue = head.value;
    }
    return isBST(head.right);
}
```

**如何判断一颗二叉树是完全二叉树**

```java
//宽度遍历二叉树：1）任意节点有右无左false 2）如果第一次遇到左右孩子不全，那么之后所有的节点都必须是叶节点 
public static boolean isCBT(Node head){
    if(head == null){
        return true;
    }
    LinkedList<Node> queue = new LinkedList<>();
    boolean leaf = false;
    Node l = null;
    Node r = null;
    queue.add(head);
    while(!queue.isEmpty()){
        head = queue.poll();
        l = head.left;
        r = head.right;
        if((leaf&&(l!=null||r!=null))||(l==null&&r!=null)){
            return false;
        }
        if(l!=null){
            queue.add(l);
        }
        if(r!=null){
            queue.add(r);
        }
        if(l==null||r==null){
            leaf=true;
        }
    }
    return true;
}
```

**如何判断一颗二叉树是否是满二叉树**

```java
//满二叉树，某层2^n-1节点数
public class Info{
    int height;
    int nodes;
    public info(int h,int n){
        height = h;
        nodes = n;
    }
}
public Info process(Node x){
    if(null == x){
        return new Info(0,0);
    }
    Info left = process(x.left);
    Info right = process(x.right);
    int height = Math.max(left.height,right.height)+1;
    int nodes = left.nodes+right.nodes+1;
    
    return new Info(height,nodes);
}
public boolean isFST(Node head){
    if(null == head){
        return true;
    }
    Info info = process(head);
    return (1<<info.height)-1==nodes?true;false;
}
```

**如何判断一颗二叉树是否是平衡二叉树**

```java
//树形DP套路！！！
public static boolean isBanlanced(Node head){
    return process(head).isBalanced;
}
public static class ReturnType{
    public boolean isBalanced;
    public int height;
    public ReturnType(boolean isB,int hei){
        isBalanced=isB;
        height=hei;
    }
}
public static ReturnType process(Node x){
    if(x==null){
        return new ReturnType(true,0);
    }
    ReturnType leftData=process(x.left);
    ReturnType rightData=process(x.right);
    int height=Math.max(leftData.height.rightData.height)+1;
    boolean isBanlanced=leftData.isBanlanced&&rightData.isBalanced
        &&Math.abs((leftData.height-rightData.height)<2);
    return new ReturnType(isBalanced,height);
}
```

**给定两个二叉树的节点node1和node2，找他们的最低公共祖先节点**

```java
//o1、o2一定属于head为头的数
//返回o1和o2最低公共祖先
//第一种算法
public static Node lowestAncestor(Node head,Node o1,Node o2){
    if(head==null||head == o1||head==o2){
        return head;
    }
    Node left = lowestAncestor(head.left,o1,o2);
    Node right = lowestAncestor(head.right,o1,o2);
    if(left!=null&&right!=null){
        return head;
    }
    //左右两棵树，并不都有返回值
    return left!=null?left:right;
}
//遍历树，将树的父节点放入map中。将o1的父节点链放入set中，然后依次遍历02的父节点，直到第一次出现在o1的父节点set中，那么该节点就是最低公告祖先节点
public static Node lca(Node head,Node o1,Node o2){
    HashMap<Node,Node>() parentMap = new HashMap<>();
    parentMap.put(head,head);
    process(head,parentMap);
    HashSet<Node> set = new HashSet<>();
    Node cur = o1;
    while(cur != head){
        set.put(cur);
        cur = parentMap.get(cur);
    }
    set.add(head);
    cur = o2;
    while(!set.contains(cur)){
        cur = parentMap.get(cur);
    }
    return cur;
}
public static void process(Node head,HashMap<Node,Node> map){
    if(head == null){
        return;
    }
    map.put(head.left,head);
    map.put(head.right,head);
    process(head.left,map);
    process(head.right,map);
}

```

**在二叉树中找到一个节点的后继节点**

【题目】现在有一种新的二叉树节点类型如下

```java
public calss Node{
    public int value;
    public Node left;
    public Node right;
    public Node parent;
    public Node(int val){
        this.value = val;
    }
}
```

**该结构比普通二叉树节点结构多了一个指向父节点的parent指针。**

**假设有一颗Node类型的节点组成的二叉树，树中每个节点的parent指针都正确的指向自己的父节点，头节点的parent指向null。**

**只给一个在二叉树中的某个节点node，请实现返回node的后继节点的函数。**

**在二叉树的中序遍历的序列中，node的下一个节点叫做node的后继节点。**

```java
//1)x有右树，x的后继是右树最左节点 2）x无右树找到第一个左树的父节点，如果没有就返回空
public static getSuccessorNode(Node node){
    if(node == null){
        return node;
    }
    if(node.right!=null){
        return getLeftMost(node.right);
    }else{
        Node parent = node.parent;
        while(parent!=null&&parent.left!=node){
            node = parent;
            parent = node.parent;
        }
        return parent;
    }
}
public static Node getLeftMost(Node node){
    if(node == null){
        return node;
    }
    while(node.left!=null){
        node = node.left;
    }
    return node;
}
```

———————————————————————————————————————————

**二叉树的序列化和反序列化**

**就是内存里的一棵树如何变成字符串形式，又如何从字符串形式变成内存里的树**

```java
//先序序列化
public static String serialByPre(Node head){
    if(head==null){
        return "#_";
    }
    String res = head.value+"_";
    res +=serialByPre(head.left);
    res +=serialByPre(head.right);
    return res
}
//先序反序列化
public static Node reconByPreString(String preStr){
    String[] value = preStr.split("_");
    Queue<String> queue= new LinkedList<>();
    for(int i=0;i<value.length;i++){
        queue.add(value[i]);
    }
    return reconPreOrder(queue);
}
public static Node reconPreOrder(Queue<String> queue){
    String value = queue.poll();
    if(value.equals("#")){
        return null;
    }
    Node head = New Node(Integer.valueof(value));
    head.left = reconPreOrder(queue);
    head.right = reconPreOrder(queue);
    return head;
}
```

**如何判断一颗二叉树是不是另一颗二叉树的子树？**

———————————————————————————————————————————

**折纸问题**

**请把一段纸条竖着放在桌子上，然后从纸条的下边向上方对折1次，压出折痕后展开。**

**此时折痕是凹下去的，即折痕突起的方向指向纸条的背面。**

**如果从纸条的下边向上方连续对折2次，压出折痕后展开，此时有三条折痕，从**

**上到下依次是下折痕、下折痕和上折痕。**

**给定一个输入参数N，代表纸条都从下边向上方连续对折N次。**

**请从上到下打印所有折痕的方向。**

**例如:N=1时，打印:downN=2时，打印:down down up**

```java
public static void printAllFolds(int N){
    printProcess(1,N,true);
}
//递归过程，来到了某一节点
//i是节点的层数，N是一共的层数，down==true凹 down==false凸
public static void printProcess(int i,int N,boolead down){
    if(i>N){
        return;
    }
    printProcess(i+1,N,true);
    System.out.println(down?"凹":"凸");
    printProcess(i+1,N,false);
}
```

