---
title: "Trie"
tags:
- atom
---
Topics:  [[Coding techniques]]
Reference:  

---

### Intuitions  
- Think of words as trees that branch out  
- As soon as you see prefix, or some continuous part of a word, tries might be handy.  

### Characteristics  
- If you have created a trie from multiple words, the longest common prefix for all the words is
the deepest node where all nodes in the path only have one child.
- Can be navigated like a tree i.e. dfs, bfs etc
- PreOrder traversal will traverse words alphabetically

### Miscellaneous  
- Can be optimised by making each node hold a the string or empty if the node is not a word.  
