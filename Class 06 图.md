# Class 06 图

**图的存储方式**

1. 邻接表:以点为单位，写出相邻的点

2. 邻接矩阵：以边为单位，矩形一定是正方形，表示点与点的距离

**如何表达图?生成图?**

```java
public class Graph{
    public HashMap<Integer,Node> nodes;
    public HashSet<Edge> edges;
    public Graph(){
        nodes = new HashMap<>();
        edges = new HashSet<>();
    }
}
public class Node {
    public int in;//入度，多少边指向它
    public int out;//从他出去多少边
    public ArrayList<Node> nexts;//点发散出去邻接点
    public ArrayList<Edge> edges;//发散出去的边
    public Node(int value){
        this.value = value;
        in = 0;
        out = 0;
        nexts = new ArrayList<>();
        edges = new ArrayList<>();
    }
}
public class Edge{
    public int weight;
    public Node form;
    public Node to;
    public Edge(int weight,Node form,Node to){
        this.weight = weight;
        this.from = from;
        this.to = to;
    }
}
//[5,0,1]
//[3,1,2]
//[7,0,2]
public static Graph createGraph(Integer[][] matrix){
    Graph graph = new Graph();
    for(int i=0;i<matrix.length;i++){
        Integer from = matrix[i][0];
        Integer to = matrix[i][1];
        Integer weight = matrix[i][2];
        if(!graph.nodes.containsKey(from)){
            graph.nodes.put(from,new Node(from));
        }
        if(!graph.nodes.containsKey(to)){
            graph.nodes.put(to,new Node(to));
        }
        Node fromNode = graph.nodes.get(from);
        Node toNode=graph.nodes.get(to);
        Edge newEdge = new Edge(weight,fromNode,toNode);
        fromNode.nexts.add(toNode);
        fromNode.out++;
        toNode.in++;
        fromNode.edges.add(newEdge);
        graph.edges.add(newEdge);
    }
    return graph;
}
```

**图的宽度优先遍历**

1. 利用队列实现
2. 从元结点开始依次按照宽度进队列，然后弹出
3. 每弹出一个点，把该节点所有没有进过队列的邻接点放入队列
4. 直到队列变空

广度优先遍历

1. 利用栈实现
2. 从源节点开始把节点按照深度放入栈，然后弹出
3. 每弹出一个点，把该节点的下一个没有进过栈的邻接点放入栈
4. 直到栈变空

```java
public class Code01_BFS{
    //从node出发，进行宽度优先遍历
    public static void bfs(Node node){
        if(node == null){
            return;
        }
        Queue<Node> queue = new LinkedList<>();
        HashSet<Node> set = new HashSet<>();
        queue.add(node);
        set.add(node);
        whie(!queue.isEmpty()){
            Node cur = queue.poll();
            System.out.println(cur.value);
            for(Node next:cur.nexts){
                if(!set.contains(next)){
                    set.add(next);
                    queue.add(next);
                }
            }
        }
    }
}
```

```java
public class Code02_DFS {
    public static void dfs(Node node){
        if(node == null){
            return;
        }
        Statck<Node> stack = new Stack<>();
        HashSet<Node> set = new HashSet<>();
        stack.add(node);
        set.add(node);
        System.out.println(node.value);
        while(!stack.isEmpty()){
            Node cur = stack.pop();
            for(Node next:cur.nexts){
                if(!set.cantains(next)){
                    stack.push(cur);
                    stack.push(next);
                    set.add(next);
                    System.out.println(next.value);
                    break;
                }
                
            }
        }
    }
}
```

**拓朴排序算法**

**适用范围：要求有向图，且有入度为0的节点，且没有环**

```java
public class Code03_TopologySort{
    //directed graph and no loop
    public static List<Node> sortedTopology(Graph graph){
        //key:某个node
        //value:剩余的入度
        HashMap<Node,Integer> inMap = new HashMap<>();
        //入读为0的点，才能进这个队列
        Queue<Node> zeroInQueue = new LinkedList<>();
        for(Node node:graph.nodes.values()){
            inMap.put(node,node.in);
            if(node.in==0){
                zeroInQueue.add(node);
            }
        }
        //拓朴排序的结果，依次加入result
        List<Node> result = new ArrayList<>();
        while(!zeroInqueue.isEmpty()){
            Node cur = zeroInQueue.poll();
            result.add(cur);
            for(Node next:cur.nexts){
                inMap.put(next,inMap.get(next)-1);
                if(inMap.get(next) == 0){
                    zeroInQueue.add(next);
                }
            }
        }
        return result;
    }
}
```

**kruskal算法**
**适用范围：要求无向图**

```java
public class Code04_Kruskal{
    public static class MySets{
        public HashMap<Node,List<Node>> setMap;
        public Myset(List<Node> nodes){
            for(Node cur:nodes){
                List<Node> set = new ArrayList<>();
                set.add(cur);
                setMap.put(cur,set);
            }
        }
    }
    
    public boolean isSameSet(Node from,Node to){
        List<Node> fromSet = setMap.get(from);
        List<Node> toSet = setMap.get(to);
        return fromSet == toSet;
    }
    
    public void union(Node from,Node to){
        List<Node> fromSet = setMap.get(from);
        List<Node> toSet = setMap.get(to);
        
        for(Node cur: toSet){
            fromSet.add(toNode);
            setMap.put(toNode,fromSet);
        }
    }
    
    public static Set<Edge> kruskalMST(Graph graph){
        UnionFind unionFind = new UnionFind();
        unionFind.makeSets(graph.nodes.values);
        PriorityQueue<Edge> priorityQueue = new PriorityQueue<>(new EdgeComparator());
        for(Edge edge: graph.edges){
            priorityQueue.add(edge);
        }
        Set<Edge> res = new HashSet<>();
        while(!priorityQueue.isEmpty()){
            Edge edge = priorityQueue.poll();
            if(!unionFind.isSameSet(edge.from,edge.to)){
                res.add(edge);
                unionFind.union(edge.from,edge.to)
            }
        }
    }
}
```

**prim算法**

**适用范围：要求无向图**

```java
public class Code04_primMST{
    public static Set<Edge> primeMST(Graph graph){
       //解锁的边进入小根堆
        PriorityQueue<Edge> priorityQueue = new PriorityQueue<>(new EdgeComparator());
        HashSet<Node> set = new HashSet<>();
        Set<Edge> result = new HashSet<>();//依次挑选的边在result里
        for (Node node: graph.nodes.values()){//随便挑了一个点
            //node是开始点
            if(!set.contains(node)){
                set.add(node);
                for(Edge edge:node.edges){
                    priorityQueue.add(edge);
                }
                while(!priorityQueue.isEmpty()){
                    Edge edge = priorityQueue.poll();//弹出解锁的边中，最小的边
                    Node toNode = edge.to;//可能的一个新的点
                    if(!set.contains(toNode)){//不含有的时候就是新的点
                        set.add(toNode);
                        result.add(edge);
                        for(Edge nextEdge:toNode.edges){
                            priorityQueue.add(nextEdge);
                        }
                    }
                }
            }
            
        }
        return result;
    }
}
```

**Dijkstra算法**

规定一个出发点，到其他点的距离，到不了就是无穷

**适用范围：没有权值为负数的边**

```java
// no negative weight
public class Code06_Dijkstra {
public static HashMap<Node, Integer> dijkstra1(Node head){
    //从head出发到所有点的最小距离
    //key:从head出发到达key
    //value:从head出发到达key的最小距离
    //如果在表中，没有T的记录，含义是从head出发到T这个点的距离为正无穷
    HashMap<Node, Integer> distanceMap = new HashMap<>();
    distanceMap.put(head,0);
    //已经求过距离的节点，存在selectedNodes中，以后再也不碰
    HashSet<Node> selectedNodes = new HashSet<>();
    Node minNode = getMinDistanceAndUnselectedNode(distanceMap, selectedNodes);
    while (minNode != null){
        int distance = distanceMap.get(minNode);
        for (Edge edge : minNode.edges){
            Node toNode = edge.to;
            if (!distanceMap.containsKey(toNode)){
                a distanceMap.put(toNode, distance + edge.weight);
            } 
            distanceMap.put(edge.to,Math.min(distanceMap.get(toNode),distance t edge.weight));
            selectedNodes.add(minNode);
            minNode = getMinDistanceAndUnselectedNode(distanceMap, selectedNodes);
        }
        return distanceMap;
    }
    
    public Static Node getMinDistanceAndUnselectedNode(HashMap<Node, Integer> distanceMap,HashSet<Node> selectedNodes){
        
    }

        
}
```

