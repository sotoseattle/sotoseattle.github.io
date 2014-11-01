---
layout: post
title: "PGN: Building Clique Trees"
date: 2013-12-14 08:01
comments: true
categories: PGN, theory
---

We define a Clique Tree (CT) as a structure with nodes (clusters of variables), edges (connections between them), and potentials (factors).

To build a CT we need a list of factors from which we start deriving the set of all random variables and the matrix of edges between them. Then we iteratively run variable elimination in the following way:

- we choose as the variable (z) to remove the first one with the least edges (min neighbors)
- we separate the factors given in two sets: one with factors that include z, and another with the rest.
- from the set of factors that include z:
	- we multiply the factors to compute the $\lambda$, 
	- create a node/cluster in the CT with all the variables of $\lambda$
	- run over all pre-existing CT nodes, and check if any of the associated $\tau$s is also a factor in the lambda product. If so, create an edge between the nodes.
	- marginalize $\lambda$ to produce $\tau$ and record it in the CT next to the cluster created.
	- add $\tau$ as a factor to the unused factor list (those that did not have z)
- Update the edges by connecting all variables inside the newly minted CT node, and removing all edges from/to the eliminated variable
- return the new factor list, edge matrix, and clique tree for another iteration.

Here is a first draft of the process:

```python Clique Tree from Factor List (unrefactored)
import numpy as np
import Factor
import Rvar
import FactorOperations

class FactorGraph(object):
    '''graph of linked random variables derived from a list of factors'''
    def __init__(self, listOfFactors):
        self.factors = listOfFactors

        # extract all variables. The index is the key for setting edges.
        cv = []
        for fu in self.factors:
            cv += fu.variables
        self.allVars = tuple(sorted(set(cv)))

        # create the adjacency matrix of the initial factor list
        numVars = len(self.allVars)
        self.edges = np.zeros((numVars, numVars)).astype('int32')
        for fu in self.factors:
            for vi in fu.variables:
                for vj in fu.variables:
                    self.edges[self.allVars.index(vi), self.allVars.index(vj)] = 1

    def firstMinNeighborVar(self):
        connections = tuple(np.sum(self.edges, axis=1))
        minE = float('+inf') 
        for e in connections:
            if e > 0 and e < minE:
                minE = e
        return self.allVars[connections.index(minE)]
    

class CliqueTree(object):
    def __init__ (self, listOfFactors):  
        F = FactorGraph(listOfFactors)
        
        # create nodes iteratively through var elim
        C = {'nodes':[], 'edges':np.zeros((0,0))}
        considered_cliques = 0
        while considered_cliques < len(F.allVars):
            z = F.firstMinNeighborVar()
            [F,C] = self.eliminateVar(F, C, z)
            considered_cliques += 1
        self.nodes = C['nodes']
        self.edges = C['edges']

    def eliminateVar(self, F, C, z):
        # separate factors into two lists, those that use z (F_Cluster) and the rest
        F_Cluster, F_Rest, Cluster_vars = [], [], []
        for f in F.factors:
            if z in f.variables:
                F_Cluster += [f]
                Cluster_vars += f.variables
            else:
                F_Rest += [f]
        
        if F_Cluster!=[]:
            Cluster_vars = tuple(sorted(set(Cluster_vars)))

            # when computing tau of new node, check if it uses other nodes' taus
            rows,cols = C['edges'].shape
            C['edges'] = np.vstack([C['edges'], np.zeros((1,cols))])
            C['edges'] = np.hstack([C['edges'], np.zeros((rows+1,1))])
            pos = np.zeros(cols+1)
            for n,node in enumerate(C['nodes']):
                if node['tau'] in F_Cluster:
                    pos[n]=1
            # create a new array of connecting node edges based on taus in common
            C['edges'][-1,:] = pos
            C['edges'][:,-1] = pos
            
            # multiply the factors in Cluster... (lambda) ...and marginalize by z (tau)
            tau = F_Cluster.pop(0)
            for f in F_Cluster:
                tau = FactorOperations.multiply(tau, f)
            if tau.variables != [z]:
                tau = FactorOperations.marginalize(tau, z)
            
            # add to unused factor list the resulting tau ==> new factor list with var eliminated
            F_Rest += [tau]
            
            # update the edges (connect all vars inside new cluster, & disconnect the eliminated variable)
            for vi in Cluster_vars:
                for vj in Cluster_vars:
                    F.edges[F.allVars.index(vi), F.allVars.index(vj)] = 1
            F.edges[F.allVars.index(z),:] = 0
            F.edges[:, F.allVars.index(z)] = 0
            
            C['nodes'] += [{'vars':Cluster_vars, 'tau':tau}]
            
            F.factors = F_Rest
        return [F, C]
```
But this is not all, we still need to prune the tree and initialize it's potentials.