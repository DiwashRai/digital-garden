---
title: "Coding challenge protocols"
tags:
-   molecule
---
Topics: [[Software Engineering]]  
Reference:  
-   [[Dynamic Programming]]
-   [[Disjoint Set]]

---

## Finding a possible solution
-   **Narrow down nature of the solution**
    -   Is it a graph problem? [[#Graph problems]]
        -   Can it be solved by considering indegrees and outdegrees?
        -   Topological sort?
        -   Disjoint set?
        -   Is the problem about connectivity?
            -   Consider Disjoint set
            -   DFS or BFS could also be more appropriate at times
    -   Is it asking you to find an 'optimum'?
        -   Can you use a greedy algorithm?
        -   Can it be represented as a series of subproblems?
            -   Try dynamic programming. Memoization or tabulation.
-   **Can "preprocessing" the data help?**
    -   If the data was sorted would it help find a solution or speed up the solution?
    -   Is there a more optimal representation for the data? e.g.
        -   Adjacency list -> Disjoint set
        -   Words list -> Trie
    -   Creating something like a prefix sum array?

## Implementing a dynamic programming algorithm
-   What is the base case?
-   Find out how many variables there are to 'cache'. i.e. do you need a 1d, 2d, 3d... array.
-   What 'calculation' or 'result' can be reused in the algorithm.
-   Is it better to go left to right or right to left?

## Graph problems
-   Ways of representing a graph/Graph data structures
    -   Adjacency Matrix
        -   e.g. `std::vector<std::vector<bool>> adjMatrix;`
    -   Adjacency list
        -   e.g. `std::unordered_map<int, std::vector<int>> adjList;`
    -   Edge list
        -   e.g. `std::vector<std::pair<int, int>> edgeList;`
    -   Incidence matrix
