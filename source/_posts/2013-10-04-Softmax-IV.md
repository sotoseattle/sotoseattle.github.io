---
layout: post
title: "Softmax Theory: $θ$"
date: 2013-10-04 09:02
comments: true
categories: regression, softmax, theory
---

To train the model and find the parameters we need some sort of gradient descent algorithm. In python [fmin_bfgs](http://en.wikipedia.org/wiki/Broyden%E2%80%93Fletcher%E2%80%93Goldfarb%E2%80%93Shanno_algorithm) takes way too long and hogs memory, while it's sister [fmin_l_bfgs_b](http://en.wikipedia.org/wiki/Limited-memory_BFGS) is much faster and lighter.

To call it we need to consider the following issues:

- we need to pass the cost and gradient function separately (unlike in Octave).
- the first argument of the function and of its derivation has to be the parameter to be optimized.
- tinit is an initial set of parameters from which to start the search, a good compromise is ```0.005 * np.random.rand(labels, n)```.
- for the l_bfgs to work the gradient format must be rolled out (not in matrix form).
- the optimizing algorithm provides additional useful information [f, d] about the search performed.
- other options, like the maximum number of iterations to perform or the convergence tolerance can be researched online.

```python optimizing thetas
import numpy as np
from scipy.optimize import fmin_l_bfgs_b

def optimizeThetas(tinit, X_train, GT, numLabels, l):
    def f(w):
        return j(w, X_train, GT, numLabels, l)
    def fprime(w):
        return v(w, X_train, GT, numLabels, l)

    [thetas, f, d] = fmin_l_bfgs_b(func=f, x0=tinit, fprime=fprime, maxiter=400)
    return thetas
```
