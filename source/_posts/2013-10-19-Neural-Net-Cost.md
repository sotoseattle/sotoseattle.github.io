---
layout: post
title: "Neural Networks Theory: Backwards"
date: 2013-10-19 09:02
comments: true
categories: neural networks, theory
---


## cost function

We use a generalization of the logistic regression cost function:

$$
\begin{align}
J(\theta) = -\frac{1}{m} \left[ \sum_{i=1}^m y^{(i)} (\log h_\theta(x^{(i)})) + (1-y^{(i)}) (\log (1-h_\theta(x^{(i)})))\right] + \frac{\lambda}{2m} \sum_{i=1}^m \theta_j^2 
\end{align}
\$$

To create our neurotic cost function:

$$
\begin{align}
J(\theta) = -\frac{1}{m} \left[ \sum_{i=1}^m \sum_{k=1}^K y_k^{(i)} \log (h_\Theta(x^{(i)}))_k + (1-y_k^{(i)}) \log (1-(h_\Theta(x^{(i)}))_k)\right] + \frac{\lambda}{2m} \sum_{l=1}^{L-1} \sum_{i=1}^s_{l} \sum_{j=1}^{s_{l+1}} {\Theta_{ji}^{(l)}}^2
\end{align}
$$

In logit there was a single output unit, so the generalized for only adds a sum over the K output units of the layer. The new regularization component is just adding over all possible values of $\Theta_{ji}^{(l)}$ for each l, j and i. Also, like logit, it excludes all the parameters for bias units.

The intuition is simple, in a fast and dirty way, the cost of example i is 'like' the square error, or squared difference between what the network says and the reality dictates.

Finally, consider that the cost function is not convex, so we can get stuck.end up in a local minimum when optimizing the parameters. In practice this is not a big problem and the usual suspects algorithms will get to a very good local minimum even if not the global one.

```python cost function
def j(thetas_rolled, layers, x, y, lam):
    m,n = x.shape
    thetas = unroll_thetas(n, layers, thetas_rolled)
    y_bloat = groundTruth(np.array(y), layers[-1])
    a, r = x, 0.
    for t in thetas:
        a = np.hstack([np.ones((m,1)),a])
        a = sigmoid(a.dot(t.T))
        r += np.sum(t[:,1:]**2)    
    J = (np.sum(-y_bloat*log(a) - (1-y_bloat)*log(1-a)) + (r*lam)/2)/m
    return J
```

## gradient

So the idea is to find the parameters $\theta$ that minimize the cost function. Like before, we need to compute both the cost and the gradient at each iteration of an optimization algorithm.

The same way that $a_j^{(l)}$ is the activation (result, output) of node j in layer l, we are going to have a $\delta_j^{(l)}$, which will represent the "error" on the activation of that node, kind of like how much is the output off from where we needed it to be.

If we go all the way to the end, we do know that the final activation is the predicted h(x) and should be equal to the 'true' observed label y. So in a case with 4 layers (4<sup>th</sup>) layer is the output layer) we have, $\delta_j^{(4)} = a_j^{(4)} - y_j = (h_{\Theta}(x))_j - y_j$ or in vectorized form: $\delta^{(4)} = a^{(4)} - y$

Now we can go backwards and compute the deltas one layer at a time considering that g' is the derivative of the activation function.

$$
\begin{align}
\delta^{(3)} = (\Theta^{(3)})^T \delta^{(4)} .* g'(z^{(3)}) = (\Theta^{(3)})^T \delta^{(4)} .* (a^{(3)} .* (1-a^{(3)}))
\\
\delta^{(2)} = (\Theta^{(2)})^T \delta^{(3)} .* g'(z^{(2)}) = (\Theta^{(2)})^T \delta^{(3)} .* (a^{(2)} .* (1-a^{(2)}))
\end{align}
$$


There is no $\delta^{(1)}$ because the first layer has no errors, is the input.

In a sense, if we forward propagate the computation of the activations in each layer from the input forward, we back propagate the computations of the errors from the output backwards.

It is possible to demonstrate mathematically that the partial derivative of the cost function is equal to $a_j^{(l)} \delta_i^{(l+1)}$ when no considering regularization.


Intuition: Imagine a node A from which two activation outputs connect it to another two nodes B and C. The $\delta$ of A is computed as a weighted average of the $\delta$ of B and C weighted by the $\Theta$ that connect them to node A.

$$
\delta_A \approx \Theta_{A \to B} \delta_B + \Theta_{A \to C} \delta_C
$$

## backpropagation algorithm

- Set $\Delta_{ij}^{(l)}$ for all l, i, j, that we will use to later compute the gradient
- For each example:
    - set $a^{(1)} = x^{(i)}$
	- forward propagate to compute all $a^{(l)}$
	- backpropagate using first $y^{(i)}$ to compute all $\delta^{(l)}$ up to $\delta^{(2)}$
	- compute the $\Delta$ updating with the partial derivative:

$$
\begin{align}
\Delta_{ij}^{(l)} &:= \Delta_{ij}^{(l)} + a_{j}^{(l)} \delta_{i}^{(l+1)}
\\
\Delta^{(l)} &:= \Delta^{(l)} + \delta^{(l+1)} (a^{(l)})^T
\end{align}
$$

- Compute D terms that correspond to the gradient, partial derivative of $J(\Theta)$ now with regularization:
	
$$
\begin{align}
D_{ij}^{(l)} &:= \frac{1}{m} \Delta_{ij}^{(l)} + \lambda \Theta_{ij}^{(l)} \mbox{  if j!=0}
\\
D_{ij}^{(l)} &:= \frac{1}{m} \Delta_{ij}^{(l)} \mbox{  if j=0}
\end{align}
$$


```python gradient function and backpropagation
def v(thetas_rolled, layers, x, y, lam):
    m,n = x.shape
    thetas = unroll_thetas(n, layers, thetas_rolled)
    num_thetas = len(layers)
    y_bloat = groundTruth(np.array(y), layers[-1])
    
    # forward => fill lists of 'activations' and 'z'
    a, z = [x], [None]
    for i in range(num_thetas):
        a[-1] = np.hstack([np.ones((m,1)),a[-1]])
        z.append(a[-1].dot(thetas[i].T))
        a.append(sigmoid(z[-1]))
    
    # backwards => fill list of deltas
    d = [(a[-1] - y_bloat)]    
    for i in range(num_thetas, 1, -1):
        d.insert(0, (d[0].dot(thetas[i-1][:,1:])) * sigmoid_gradient(z[i-1]))
    
    # forward => compute gradient
    V = np.array([])
    for i in range(num_thetas):
        Vwip = (d[i].T.dot(a[i]) + (lam*thetas[i]))/m
        Vwip[:,0] -= (lam/m) * thetas[i][:,0]
        V = np.hstack([V, Vwip.flatten()])
    return V

def optimizeThetas(tinit, layers, x, y, lam, visual=True):
    def f(w):
        return j(w, layers, x, y, lam)
    def fprime(w):
        return v(w, layers, x, y, lam)
    
    [thetas, f, d] = fmin_l_bfgs_b(func=f, x0=tinit, fprime=fprime, maxiter=50)
    if visual:
        print thetas[0:10]
        print f
        print d
    return thetas
```










