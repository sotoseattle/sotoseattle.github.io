---
layout: post
title: "Clique Tree as Graph"
date: 2014-01-18 08:01
comments: true
categories: PGN, theory
---

Now, armed with the graph code, we can recode the Clique Tree construction algorithm. The code is greatly simplified although it works along the same conceptual lines as before. We only need to add a method to compact the tree since some nodes are being voided when pruning.

```python Clique Tree
class CliqueTree(UndGraph, object):

    def __init__ (self, listOfFactors):
        super(CliqueTree, self).__init__([])
        
        # create induced markov net from list of factors
        F = IndMarkovNet(listOfFactors)
        
        # create nodes iteratively through var elimination
        self.tau = [] # wee need this to hold the taus as we compute
        considered_cliques = 0
        while considered_cliques < F.V:
            z = F.firstMinNeighborVar()
            F = self.eliminateVar(F, z)
            considered_cliques += 1
        F, self.tau = None, []
        
        # prune, compact and initialize resulting tree
        keepPruning = True
        while keepPruning:
            keepPruning = self.pruneNode()
        self.compactTree()
        self.initializePotentials(listOfFactors)
    
    def eliminateVar(self, F, z):
        # separate factors into two lists, those that use z (F_Cluster) and the rest
        F_Cluster, F_Rest, Cluster_vars = [], [], []
        for f in F.factors:
            if z in f.variables:
                F_Cluster += [f]
                Cluster_vars += f.variables
            else:
                F_Rest += [f]
        
        if F_Cluster!=[]:
            # add a node to clique tree with the variables involved
            position = self.V
            self.V += 1
            self.box.insert(position, set(Cluster_vars))
            self.adj.insert(position, [])
            
            # when computing tau of new node, check if it uses other nodes' taus and connect
            for i in range(position):
                if self.tau[i] in F_Cluster:
                    self.addEdge(i, position)
            
            # multiply the factors in Cluster... (lambda) ...and marginalize by z (tau)
            tau = F_Cluster.pop(0)
            for f in F_Cluster:
                tau = FactorOperations.multiply(tau, f, False)
            if tau.variables != [z]:
                tau = FactorOperations.marginalize(tau, z)
            self.tau.insert(position, tau)
            
            # update the edges of F (connect all vars inside new factor, & disconnect the eliminated variable)
            F.connectAll([F.index_var(v) for v in self.box[position]])
            F.adj[F.index_var(z)] = []
            
            # add to unused factor list the resulting tau ==> new factor list with var eliminated
            F_Rest += [tau]            
            F.factors = F_Rest
        return F

    def pruneNode(self):
        '''Start with a node (A), scan through its neighbors to find one that is a superset of variables (B). 
           Add edges between B and all of A's other neighbors and cut off all edges from A.'''
        for i,nodeA in enumerate(self.box):
            neighbors = self.adj[i]
            for j in neighbors:
                nodeB = self.box[j]
                if nodeA and nodeA < nodeB:
                    # connect nodeB to other nodeA's neighbors
                    for e in neighbors:
                        if j!=e:
                            self.addEdge(j,e)
                    # disconnect child
                    self.adj[i] = []
                    self.box[i] = set()
                    return True
        return False
    
    def compactTree(self):
        '''compacts the clique tree after prunning to remove null nodes'''
        translatable = {}
        counter = 0
        for i in range(self.V):
            if self.box[i]:
                translatable[i] = counter
                counter +=1
        newV = len(translatable.keys())
        nodes, lnk = [None]*newV, [None]*newV
        for k in translatable:
            nodes[translatable[k]] = self.box[k]
            lnk[translatable[k]] = [translatable[e] for e in self.adj[k] if e in translatable]
        self.V = newV
        self.box = nodes
        self.adj = lnk
        pass
    
    def initializePotentials(self, listOfFactors):
        # create factors initialized to ones
        self.factors = [None]*self.V
        for i in range(self.V):
            fu = Factor.Factor(sorted(list(self.box[i])))
            fu.values = np.ones(fu.cards)
            self.factors[i] = fu
        
        # ... and now brutishly (FIFO) we assign the factors
        for fu in listOfFactors:
            notUsed = True
            for i,n in enumerate(self.box):
                if n.issuperset(set(fu.variables)):
                    self.factors[i] = FactorOperations.multiply(self.factors[i], fu, False)
                    notUsed = False
                    break  # to use only once
            if notUsed:
                raise NameError('factor not used in any clique!', fu.variables)
        pass
```