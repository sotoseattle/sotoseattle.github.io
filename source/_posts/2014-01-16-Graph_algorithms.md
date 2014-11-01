---
layout: post
title: "Graph Algorithim"
date: 2014-01-16 08:01
comments: true
categories: algos, code
---

After a long vacation I have decided to step back and redo the code to include graph algorithims. These ideas come from the Algorithms course I took from Princeton University through Coursera.

Both, Markov networks and Clique Trees can be better manipulated in a graph structure. Benefits are the ability to query the graph about itself (does it have loops?, find efficient paths between two points, etc) and perform efficient searches. But the real reason is that I hate to use the structures inherited from the Octave code, they are too weird and ugly, with huge sparse matrices and disconnected data sub-structures. I can hold everything inside a graph in a more intuitive manner and the data is associated to the natural nodes or its edges.

I use a basic structure for the simplest graph of all, the undirected graph. It holds a list of nodes that will hold variables in a set form, and a list of edges for each node that tells us to which other nodes it connects to.

```python Undirected Graph
class UndGraph(object):
    '''Undirected Graph. We keep edges in a vertex indexed adjacency list
       From Princeton Coursera Algorithms Class'''

    def __init__(self, variable_groups):
        self.V = len(variable_groups)  # total vertices
        self.adj = [None]*self.V        # edges
        self.box = [None]*self.V        # variables
        for i,e in enumerate(variable_groups):
            self.box[i] = set(e)
        for i in range(self.V):
            self.adj[i] = []
        
    def addEdge(self, v, w):
        '''adds edge between vertices v and w'''
        if (v>=self.V) or (v<0) or (w>=self.V) or (w<0):
            raise Exception("Error: nodes out of bounds in graph", v, w, self.V)
        if v==w:
            return
        if w not in self.adj[v]:
            self.adj[v].insert(0, w)
        if v not in self.adj[w]:
            self.adj[w].insert(0, v);
        pass
            
    def index_var(self, var):
        for i in range(self.V):
            if var in self.box[i]:
                return i
        return None
        
    def connectAll(self, nodeList): # a utility method that comes up frequently
        abc = list(nodeList)
        while len(abc)>1:
            a = abc.pop()
            for b in abc:
                self.addEdge(a, b)
        pass
    
    def __str__(self):
        s = `self.V` + " vertices\n"
        for i, v in enumerate(self.box):
            s += "["+`i`+"] " + `list(v)` + ": " + `[list(self.box[e]) for e in self.adj[i]]` + "\n"
        return s
```

We also include a decoupled class for the Breadth First Search Algorithim. A search method useful in itself but that will be come handy, later on, for finding a correct message propagation path along a clique tree.

```python Breadth First Search
from UndGraph import UndGraph

class BFS_Paths(object):
    '''Breadth First Search Algorithm. Decoupling the DAG from its processing
       From Princeton Coursera Algorithms Class'''
    
    def __init__(self, G, s):
        self.G = G
        self.source = s
        self.discoveryPath = []
        self.marked = [False]*self.G.V
        self.edgeTo = [None]*self.G.V
        self.distTo = [float("inf")]*self.G.V
        self.bfs(self.source)
    
    def bfs(self, v):
        self.distTo[self.source] = 0
        self.marked[self.source] = True
        queue = [self.source]
        while len(queue)>0:
            v = queue.pop()
            self.discoveryPath.append(v)
            for w in self.G.adj[v]:
                if self.marked[w]==False:
                    self.edgeTo[w] = v
                    self.distTo[w] = self.distTo[v] + 1
                    self.marked[w] = True
                    queue.insert(0, w)
        # the reverse way of discovery is the way to pass messages
        self.discoveryPath.reverse()
        pass
    
    def distance(self, v):
        return self.distTo[v]
    
    def pathTo(self, v):
        if self.marked[v]==False:
            return None
        else:
            path = [v]
            x = v
            while self.distTo[x] != 0: 
                x = self.edgeTo[x]
                path.insert(0,x)
            return path
```
