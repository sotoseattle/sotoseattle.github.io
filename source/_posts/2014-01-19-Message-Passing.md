---
layout: post
title: "Message Passing for Belief Propagation"
date: 2014-01-19 08:01
comments: true
categories: PGN, theory
---

Since our clique tree is an undirected graph we can use the BFS (Breadth First Search) algorithm to give us a correct path for belief propagation. This will be the reversed order in which nodes are discovered in BFS. The only two caveats to consider are:

- we need to start from a leaf of the tree (which has only one neighboring node)
- the path given is the forward path, a correct backward path can be derived from reversing the forward one.

```python Message Passing Path
def computePath(self):
        start = None # choose an arbitrary leaf
        for i in range(self.V):
            if len(self.adj[i])==1:
                start = i
                break
        forward = BFS_Paths(self, start).discoveryPath
        forward.pop() # we dont need the last node
        
        temp = []
        marked = [False]*self.V
        for from_v in forward:
            # every node in the path is ensured to have received all necessary incoming messages
            neighbors = self.adj[from_v]
            marked[from_v] = True
            potentials = [n for n in neighbors if not marked[n]]
            to_v = potentials[0]
            temp.append([from_v, to_v])
        
        edges = temp
        for e in reversed(temp):    
            edges.append(e[::-1])
        return edges
```