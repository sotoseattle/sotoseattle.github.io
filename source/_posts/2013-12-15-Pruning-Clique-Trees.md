---
layout: post
title: "PGN: Pruning Clique Trees"
date: 2013-12-15 08:01
comments: true
categories: PGN, theory
---

In the previous post we created a Clique Tree from a list of factors. The problem is that many times, the clique tree will be too unnecessary big. Many cliques will be subsets of others. It makes sense to collect them together than keep them alive and repeat unnecessary motions. For example, if we have all the info in a clique with variables A,B,C we don't need another clique (node) that holds variables A and B, since the first one already has it all. We can create a more compact and efficient tree if we prune these unnecessary nodes.

The process is not complicated: Go over all the nodes of the tree one at a time. For each one get its connected nodes (neighbors) and go through them. If the variable set of the node under study is a strict subset of the variable set of one of its connected neighbors we can prune the subset node. The key step is to reconnect the superset node to all the other neighbors of the subset node. Then we just delete the subset node and its associated row and column from the edges matrix.

For example, imagine the following clique tree:

ABC --- AB --- AE

Let's say we start with AB. We scan through its neighbors (ABC, AE) and find that AB is a subset of ABC. So we cut off the edges connected to AB and add an edge between ABC and all of AB's other neighbors (in this case just AE). This maintains the running intersection property and compacts the clique tree which now looks like: ABC -- AE.

```python Pruning (unrefactored)

class CliqueTree(object):    

    def __init__ (self, listOfFactors): 
        # ... add to the end of code
        keepPruning = True
        while keepPruning:
            keepPruning = self.pruneNode()

    def pruneNode(self):
        for idx,nodeA in enumerate(self.nodes):
            neighbors = [j for j, e in enumerate(self.edges[idx,:]) if e==1]
            for neighbor_idx in neighbors:
                if nodeA < self.nodes[neighbor_idx]:  # striself variable subset
                    for k in neighbors:
                        if k!=neighbor_idx:
                            self.edges[neighbor_idx, k] = 1
                            self.edges[k, neighbor_idx] = 1
                    self.edges = np.delete(self.edges, idx, 0)  # delete row,col of edges for nodeA
                    self.edges = np.delete(self.edges, idx, 1)
                    self.nodes = np.delete(self.nodes, idx)      # delete nodeA
                    return True
        return False
```