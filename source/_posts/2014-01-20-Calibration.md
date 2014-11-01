---
layout: post
title: "Calibration of Clique Tree"
date: 2014-01-20 08:01
comments: true
categories: PGN, theory
---

Beforehand we built the clique tree (with all its nodes, edges and initialized potentials) and a sensible propagation schedule. Now, we follow the schedule and compute for each edge the message ($\delta_{ij}$) passed between nodes $C_i$ and $C_j$. After just two passes, once forward and once backwards, we are guaranteed to have achieved calibration over the junction tree. At this moment all the necessary $\delta$ are now computed and we can compute the beliefs ($\beta$) in each node as the product of the initial potential and all its incomming messages. Having achieved calibration the exact marginals of all variables coincide independently on which belief/node we marginalize.


```python Calibration
	def __init__ (self, listOfFactors):
		...
		# clone an initialized adj structure for message propagation
    	self.delta = [[None]*len(self.adj[i]) for i in range(self.V)]

    def mssg(self, from_v, to_w):
        # collect all mssg arriving at v
        mess = []
        neighbors = self.adj[from_v]
        for n in neighbors:
            if n!=to_w:
                pos = self.adj[n].index(from_v)
                msg = self.delta[n][pos]
                mess.append(msg)

        # take the the initial Psi (and log if needed)
        d = copy.copy(self.factors[from_v])
        d.values = np.log(d.values)

        # multiply/sum by incoming messages
        for ms in mess:
            d = FactorOperations.multiply(d, ms, True)

        # marginalized to setsep vars
        for n in d.variables:
            if n not in (self.box[from_v] & self.box[to_w]):
                d = FactorOperations.marginalize(d, n)
        return d

    def calibrate(self):
        self.beta = [None]*self.V
        # compute messages
        for e in self.computePath():
            from_v, to_w = e
            pos_to = self.adj[from_v].index(to_w)
            self.delta[from_v][pos_to] = self.mssg(from_v, to_w)
        
        # compute the beliefs
        for v in range(self.V):
            belief = copy.copy(self.factors[v])
            for w in self.adj[v]:
                pos = self.adj[w].index(v)
                delta = self.delta[w][pos]
                belief = FactorOperations.multiply(belief, delta, False)
            self.beta[v] = belief
```


