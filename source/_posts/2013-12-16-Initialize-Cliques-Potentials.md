---
layout: post
title: "PGN: Initialize Clique Potentials"
date: 2013-12-16 08:01
comments: true
categories: PGN, theory
---

The goal is to assign the factors from the initial factor list to the cliques (nodes) in the Clique Tree. We do it in two steps.

First, we create factors/potentials for each node, with all values initialized to one. Then for each factor in the input factor list, we choose the first node whose variable set holds all the factor's variables, and the node's potential becomes the factor product of the node's potential itself and the newly assigned factor (remember that we set an innocuous factor of ones beforehand). 

If a factor is unassigned we raise an exception. 

If a node is factor-less (or one of its variables), there is no problem because we had it already initialized to ones.

```python Initializing Potentials (unrefactored)
class CliqueTree(object):

    def __init__ (self, listOfFactors):
        # ... add to the end of code
        # initialize all potentials first to ones
        self.factors = []
        for i in range(len(self.nodes)):
            fu = Factor.Factor(sorted(list(self.nodes[i])))
            fu.values = np.ones(fu.cards)
            self.factors += [fu]
        # ... and now brutishly (FIFO) we assign the factors
        for fu in listOfFactors:
            notUsed = True
            for i in [i for i,n in enumerate(self.nodes) if set(fu.variables) <= n]:
                self.factors[i] = FactorOperations.multiply(self.factors[i], fu, False)
                notUsed = False
                break  # to use only once
            if notUsed:
                raise NameError('factor not used in any clique!', fu.variables)
```

