---
layout: post
title: "Induced Markov Network"
date: 2014-01-17 08:01
comments: true
categories: PGN, theory
---

As the first step to build the clique tree we need to create an induced Markov graph from the factors given. We moralize by connecting all the variables in each factor (and therefore losing conditional independences in the process).

We also include a small method to return an optimal node from the graph for variable elimination. In this case we look for the first node in the graph that has the lowest number of neighboring nodes. Similar methods can be coded for different criteria depending on the way we want to build a clique tree through variable elimination.

```python Induced Markov Network
class IndMarkovNet(UndGraph, object):
    '''Induced Markov Network as an undirected graph of linked random variables
       derived from a list of factors'''
    
    def __init__(self, listOfFactors):
        set_vars = sorted(set(v for fu in listOfFactors for v in fu.variables))
        super(IndMarkovNet, self).__init__([[e] for e in set_vars])
        self.factors = listOfFactors
        for fu in self.factors:
            self.connectAll([self.index_var(v) for v in fu.variables])
    
    def firstMinNeighborVar(self):
        minino, pos = float('inf'), None
        for i,a in enumerate(self.adj):
            if 0<len(a)<minino:
                minino = len(a)
                pos = i
        return list(self.box[pos])[0]
```