# UnionFind

> 并查集结构主要用来解决“动态连通性”的问题

```
class UF {
    // 连通分量个数
    private int count;
    // 存储每个节点的父节点
    private int[] parent;
    
    public UF(int n) {
        this.count = n;
        parent = new int[n];
        for (int i = 0; i < n; ++i) {
            // 初始化的时候一个节点的父节点就是自己本身
            parent[i] = i;
        }
    }
    
    // 连通两个节点
    public void connect(int p, int q) {
        int rootP = find(p);
        int rootQ = find(q);
        if (rootP == rootQ) {
            return;
        }
        parent[rootP] = rootQ;
        --count;
    }
    
    // 找到一个节点的父节点
    public int find(int x) {
        // 注意这里是if不是while
        if (parent[x] != x) {
            parent[x] = find(parent[x]);
        }
        return parent[x];
    }
    
    // 判断节点是否连通
    public boolean connected(int p, int q) {
        return find(p) == find(q);
    }
}
```
