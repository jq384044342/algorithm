# Class 04 链表

​	**时间复杂度、空间复杂度、和稳定性**

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

**哈希表的简单介绍**

1. 哈希表在使用层面上可以理解为一种集合结构
2. 如果只有key，没有伴随数据value，可以使用HashSet结构（C++中交UnOrderedSet)
3. 如果既有key，又有伴随数据value，可以使用HashMap结构（C++中交UnOrderedMap)
4. 有无伴随数据，是HashMap和HashSet的唯一区别，底层是同一结构
5. 使用哈希表增删改查的操作，可认为时间复杂度都是O(1),但是常数时间比较大
6. 放入哈希表的数据，如果是基础类型，内部按值传递，内存占用就是这个东西的大小
7. 放入哈希表的数据，如果不是基础类型，内部按引用传递，内存占用就是这个东西内存地址的大小

**有序表的简单介绍**

1. 有序表在使用层面上可以理解为一种集合结构
2. 如果只有key，没有伴随数据value，可以使用TreeSet结构
3. 如果既有key，又有伴随数据value，可以使用TreeMap结构
4. 有无伴随数据是TreeSet和TreeMap的唯一区别，底层的实际结构是一回事
5. 有序表和哈希表的区别是，有序表把key按照顺序组织起来，而哈希表完全不组织
6. 红黑树、AVL树、size-balance-tree和跳表属于有序表结构，只是底层具体实现不同
7. 放入哈希表的数据，如果是基础类型，内部按值传递，内存占用就是这个东西的大小
8. 放入哈希表的数据，如果不是基础类型，内部按引用传递，内存占用就是这个东西内存地址的大小
9. 不管是什么底层具体实现，只要是有序表，都有以下固定的基本功能和固定的时间复杂度

**单链表的节点结构**

```java
Class Node<V>{
    V value;
    Node next;
}
```

**由以上结构的节点依次连接起来所形成的链叫单链表结构**

**双链表的节点结构**

```java
Class Node<V>{
    V value;
    Node next;
    Node last;
}
```

**由以上结构的节点依次连接起来所形成的链叫双链表结构**

**单链表和双链表结构只需要给定一个头部节点head，就可以找到剩下所有的节点**

**Question反转单向和双向链表**

【题目】分别实现反转单向链表和反转双向链表的函数

【要求】如果链表长度为N，时间复杂度要求为O(N)，额外空间复杂度要求为O(1)

```java
public static Node<V> reverse(Node<V> head){
    
}
```

**打印两个有序链表的公共部分：给定两个有序链表的头指针head1和head2，打印两个链表的公告部分。如果两个链表的长度之和为N，时间复杂度要求为O(N),额外空间复杂度要求为O(1)**

```java
//双指针移动
public static void print(Node<V> head1,Node<V> head2){
    Node one = head1;
    Node two = head2;
}
```

**面试时链表解题的方法论**

1. 对于笔试，不用太在乎空间复杂度，一切为了时间复杂度
2. 对于面试，时间复杂度依然放在第一位，但是一定要找到空间最省的方法

**重要技巧**

1. 二外数据结构记录（哈希表等）
2. 快慢指针

**判断一个链表是否为会问结构**
【题目】 给定一个单链表的头节点head，请判断该链表是否为回文结构。

【例子】1->2->1,返回true;1->2->2->1,返回true；15->6->15，返回true；1->2->3，返回false；

【要求】如果链表长度为N，时间复杂度达到O(N),额外空间复杂度达到O(1)。

```java
//快慢指针，回文结构是中间对称的
```

**将单向链表按某值划分成左边小、中间相等、右边大的形式**

[题目]给定一个单链表的头节点head，节点的值类型是整型，再给定一个整数pivot。实现一个调整链表的函数，将链表调整为左部分都是值小于pivot的节点，中间部分都是值等于pivot的节点，右部分都是值大于pivot的节点。

[进阶]在实现原问题功能的基础上增加如下的要求

[要求]调整后所有小于pivot的节点之间的相对顺序和调整前一样

[要求]调整后所有等于pivot的节点之间的相对顺序和调整前一样

[要求]调整后所有大于pivot的节点之间的相对顺序和调整前一样

[要求]时间复杂度请达到0(N)，额外空间复杂度请达到0(1)。

```java
//给6个节点，分别代表小于区域的头尾，等于区域的头尾，大于区域的头尾，然后将小于尾链接等于头，等于尾链接大于头即可。注意边界问题，有可能某一区域为空
public static Node pivot(Node head,int pivot){
    if(null == head||null == head.next){
        return;
    }
    Node sH=null;
    Node sT=null;
    Node eH=null;
    Node eT=null;
    Node mH=null;
    Node mT=null;
    Node node = head;
    while(node!=null){
        if(node.value<pivot){
            if(sH==null){
                sH=node;
                sT=node;
            }else{
                sT.next = node;
                sT=node;
            }
        }else if(node.value==pivot){
            if(eH==null){
                eH=node;
                eT=node;
            }else{
                eT.next=node;
                eT=node;
            }
        }else{
            if(mH==null){
                mH=node;
                mT=node;
            }else{
                mT.next=node;
                mT=node;
            }
        }
        node = node.next();
    }
    if(sT!=null){
        st.next = eh;
        eT=eT==null?sT:eT;
    }
    if(eT!=null){
        eT.next=mH;
    }
    return sH!=null?sH:(eH!=null?eH:mH);
}
```

**复制含有随机指针节点的链表**

【题目】一种特殊的单链表节点描述如下

```java
class Node{
    int value;
    Node next;
    Node rand;
    Node (int val){
        value = val;
    }
}
```

rand指针是单链表节点结构中新增的指针，rand可能指向链表中的任意一个节点，也可能指向null。给定一个由Node节点类型组成的无环单链表的头节点head，请实现一个函数完成这个链表的复制，并返回复制的新链表的头节点。

【要求】时间复杂度O(N)，额外空间复杂度O(1)

```java
//如果用额外空间，遍历2边，第一遍复制，第二部维持节点关系
public static Node copy1(Node head){
    if(head==null){
        return null;
    }
    new HashMap<Node,Node> map=new HashMap<>();
    Node cur = head;
    while(cur!=null){
        map.put(head,new Node(head.value));
        cur = cur.next;
    }
    cur = head;
    while(cur!=null){
        map.get(cur).next=map.get(cur.next);
        map.get(cur).rand=map.get(cur.rand);
    }
    return map.get(head);
    
}
//不用哈希表
//在原链表的基础上，让每个节点的next指向自己的克隆节点
//那么克隆节点的next，就是原链表next的next
//克隆节点的rand，就是原链表的rand的next
//然后剥离出克隆节点就好
public static Node copy2(Node head){
    if(head==null){
        return null;
    }
    Node cur=head;
    Node next=null;
    while(cur!=null){
        next= cur.next;
        cur.next = new Node(cur.value);
        cur.next.next=next;
        cur=next;
    }
    cur = head;
    Node curCopy=null;
    while(cur!=null){
        next = cur.next.next;
        curcopy = cur.next;
        curCopy.rand = cur.rand!=null?cur.rand.next:null;
        cur = next;
    }
    Node res = head.next;
    cur = head;
    //split
    while(cur!=null){
        next = cur.next.next;
        curCopy= cur.next;
        cur.next=next;
        curCopy.next=next!=null?next.next:null;
        cur=next;
    }
    return res;
}
```

**Question**

两个单链表相交的一系列问题

【题目】给定两个可能有环也可能无环的单链表，头节点head1和head2。请实现一个函数，如果两个链表相交，请返回相交的第一个节点。如果不相交，返回null

【要求】如果两个链表长度之和为N，时间按复杂度请达到O(N)，额外空间复杂度请达到O(1)

```java
//有环无环的情况不一样，所以先判断链表是否有环，并找出第一个入环节点
//额外空间可以使用HashSet，遍历链表每放入一个节点，就判断set中是否存在，存在就是有环且是第一个入环节点
//快慢指针，都从头节点出发，快走2步，慢走1步，如果快走到null，说明无环，如果快慢相遇，说明有环，并且快指针环内走的圈数不会超过2圈。相遇后，快指针放到head，慢指针不动，都每次走一步，相遇时就是入环节点。
public static Node getLoopNode(Node head){
    if(null == head||null == head.next||null==head.next.next){
        return null;
    }
    Node fast = head.next.next;
    Node slow = head.next;
    while(fast !=slow){
        if(null==fast.next||null == fast.next.next){
            return null;
        }
        fast = fast.next.next;
        slow = slow.next;
    }
    fast = head;
    while(fast!=slow){
        fast = fast.next;
        slow = slow.next;
    }
    return fast;
}
//如果两个链表都无环，返回相交节点
public static Node noLoopNode(Node head1,Node head2){
    if(null == head1||null == head2){
        return null;
    }
    Node cur1 = head1;
    Node cur2 = head2;
    int n =0;
    while(cur1.next!=null){
        cur1 = cur1.next;
        n++;
    }
    while(cur2.next!=null){
        cur2 = cur2.next;
        n--;
    }
    //结尾不相等必不相交
    if(cur1!=cur2){
        return null;
    }
    cur1 = n>0?head1:head2;
    cur2 = cur1==head1?head2:head1;
    n = Math.abs(n);
    while(n!=0){
        n--;
        cur1 = cur1.next;
    }
    while(cur1!=cur2){
        cur1=cur1.next;
        cur2=cur2.next;
    }
    return cur1;
}
//
public static Node (Node head1,Node head2){
    
}

```

