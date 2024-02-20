---
title: "Disjoint Set"
tags:
- atom
---
Topics: [[Coding techniques]]  
Reference:  

---

# Intro
Given vertices and edges between them, the Disjoint Set data structure allows us to check whether
two vertices are connected efficiently. It is also known as the ==union-find== data structure and
might also be referred to as an algorithm.

### Terminology
- Parent node: The direct parent of a node.
- Root node: A node without a parent i.e. when a node is it's own parent.

### Implementing a disjoint set
- The ==find== function: Finds the root node of a given vertex.
- The ==union== function: Unions/merges two vertices and makes their root nodes the same.
- Array of root nodes.

### Two alternative ways to implement a disjoint set
- **Implementation with Quick Find**: The time complexity of ==find== will be O(1), but the ==union==
function will take more time with O(N).
- **Implementation with Quick Union**: Compared to quick find, the complexity of ==union== is better
but the ==find== function will take longer.

# Quick Find - Disjoint Set

The basic premise is that there will be an extra step in the **union** function. After unioning two
nodes, you have to iterate through the ==root== array and change the root node of all the child nodes
of the node that was made a child into the node that was made the parent.

## Example implementation

```cpp
class UnionFind {
public:
    UnionFind(int sz) : root(sz) {
        for (int i = 0; i < sz; i++) {
            root[i] = i;
        }
    }

    int find(int x) {
        return root[x];
    }

    void unionSet(int x, int y) {
        int rootX = find(x);
        int rootY = find(y);
        if (rootX != rootY) {
            for (int i = 0; i < root.size(); i++) {
                if (root[i] == rootY) {
                    root[i] = rootX;
                }
            }
        }
    }

    bool connected(int x, int y) {
        return find(x) == find(y);
    }

private:
    vector<int> root;
};
```

## Time complexity
- Union-find constructor: O(N)
- Find: O(1)
- Union: O(N)
- Connected: O(1)

## Space complexity
- O(N) for the array.

# Quick Union - Disjoint Set
The main difference is that the array of root nodes will no longer be kept 'flat'. In fact,
it might be more accurate to call it the parent array now. This means the find function will
need to recurse through the array to find the actual root. On the other hand, the union function
will now not include that extra step to clean up old references  to the root to the new correct
ones potentially making it faster.

## Example implementation

```cpp
class UnionFind {
public:
    UnionFind(int sz) : root(sz) {
        for (int i = 0; i < sz; i++) {
            root[i] = i;
        }
    }

    int find(int x) {
        while (x != root[x]) {
            x = root[x];
        }
        return x;
    }

    void unionSet(int x, int y) {
        int rootX = find(x);
        int rootY = find(y);
        if (rootX != rootY) {
            root[rootY] = rootX;
        }
    }

    bool connected(int x, int y) {
        return find(x) == find(y);
    }

private:
    vector<int> root;
};

```

## Time complexity
- Constructor: O(N)
- Find: O(N)
- Union: O(N)
- Connected: O(N)

## Quick Find vs Quick Union
In general the Quick Union implementation is more efficient. This might be surprising as the
time complexity of the functions seem higher. However, the find function is O(N) only in the
worst case scenario. The **union** and **connected** operations have O(N) as they are dependent on the
**find** function. Contrast this to the union function for Quick Find where the union **always**
takes O(N) meaning that connecting N elements results in O(N^2) time complexity. However, Quick
Union is only O(N^2) in the worst case scenario.

# Union by Rank - Disjoint Set
The concerning inefficiency for both implementations so far is that the **union** operation always
takes O(N) time. This can be optimised by by unioning by rank. This means ordering by a
specific criteria. Previously, we have always been selecting the `X` node as the parent (the first
arg). However, we can limit the maximum height of the vertex by choosing the parent node based
on a certain criteria.

We can choose the root node to be the one with the larger "rank". This means merging the shorter
tree under the taller tree. This allows us to avoid connecting all vertices as a straight line.

This optimisation is called =="disjoint set" with union by rank==. Since this optimisation focuses
on the union function, this optimisation is only applicable for **quick union**.

## Sample Implementation
```cpp
class UnionFind {
public:
    UnionFind(int sz) : root(sz), rank(sz) {
        for (int i = 0; i < sz; i++) {
            root[i] = i;
            rank[i] = 1;
        }
    }

    int find(int x) {
        while (x != root[x]) {
            x = root[x];
        }
        return x;
    }

    void unionSet(int x, int y) {
        int rootX = find(x);
        int rootY = find(y);
        if (rootX != rootY) {
            if (rank[rootX] > rank[rootY]) {
                root[rootY] = rootX;
            } else if (rank[rootX] < rank[rootY]) {
                root[rootX] = rootY;
            } else {
                root[rootY] = rootX;
                rank[rootX] += 1;
            }
        }
    }

    bool connected(int x, int y) {
        return find(x) == find(y);
    }

private:
    vector<int> root;
    vector<int> rank;
};
```

## Time complexity
- Constructor: O(N)
- Find: O(logN)
- Union: O(logN)
- Connected: O(logN)


# Path compression optimisation - Disjoint Set
The premise of this optimisation is to update the parent node of all traversed elements when
find is called and updating their value to the root node as well as returning the root node.

## Sample implementation
In particular, pay attention to the find function.
```cpp
class UnionFind {
public:
    UnionFind(int sz) : root(sz) {
        for (int i = 0; i < sz; i++) {
            root[i] = i;
        }
    }

    int find(int x) {
        if (x == root[x]) {
            return x;
        }
        return root[x] = find(root[x]);
    }

    void unionSet(int x, int y) {
        int rootX = find(x);
        int rootY = find(y);
        if (rootX != rootY) {
            root[rootY] = rootX;
        }
    }

    bool connected(int x, int y) {
        return find(x) == find(y);
    }

private:
    vector<int> root;
};
```

## Time complexity
- Constructor: O(N)
- Find: O(logN)
- Union: O(logN)
- Connected: O(logN)

# Path compression + Union by rank sample implementation
```cpp
class UnionFind {
public:
    UnionFind(int sz) : root(sz), rank(sz) {
        for (int i = 0; i < sz; i++) {
            root[i] = i;
            rank[i] = 1;
        }
    }

    int find(int x) {
        if (x == root[x]) {
            return x;
        }
        return root[x] = find(root[x]);
    }

    void unionSet(int x, int y) {
        int rootX = find(x);
        int rootY = find(y);
        if (rootX != rootY) {
            if (rank[rootX] > rank[rootY]) {
                root[rootY] = rootX;
            } else if (rank[rootX] < rank[rootY]) {
                root[rootX] = rootY;
            } else {
                root[rootY] = rootX;
                rank[rootX] += 1;
            }
        }
    }

    bool connected(int x, int y) {
        return find(x) == find(y);
    }

private:
    vector<int> root;
    vector<int> rank;
};
```

