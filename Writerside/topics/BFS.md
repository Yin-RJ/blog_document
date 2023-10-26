# BFS

BFS问题的本质就是在一幅图中找到从起点到终点的最近距离。
```
int BFS(Node start, Node target) {
    Queue<Node> queue;
    Set<Node> visited;
    queue.add(start);
    visited.add(start);
    int step = 0;
    
    while(!queue.isEmpty()) {
        int size = queue.size();
        for (int i = 0; i < size; ++i) {
            Node node = queue.poll();
            if (node is target) {
                return step;
            }
            
            // node.adj()表示与node相邻的节点
            for (Node x : node.adj()) {
                if (x not in visited) {
                    queue.add(x);
                    visited.add(x);
                }
            }
        }
        step++;
    }
}
```