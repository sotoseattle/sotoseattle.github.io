---
layout: post
title: "Softmax Theory:  $J(θ)$"
date: 2013-10-02 09:02
comments: true
categories: regression, softmax, theory
---


The cost function of logistic regression was defined as:

$$
\begin{align}
J(\theta)
&= - \frac{1}{m} \left[ \sum_{i=1}^{m} \sum_{j=0}^{1} 1\left\{y^{(i)} = j\right\} \log p(y^{(i)} = j | x^{(i)} ; \theta) \right]
\end{align}
$$


Softmax is just a generalization of logistic regression from 2 to k classes. In our case we have:

$$
p(y^{(i)}=j | x^{(i)} ; \theta) = \frac{e^{\theta_j^T x^{(i)}}}{\sum_{l=1}^k e^{ \theta_l^T x^{(i)}} }
$$


And the softmax cost function is

$$
\begin{align}
J(\theta) = - \frac{1}{m} \left[ \sum_{i=1}^{m} \sum_{j=1}^{k} 1\left\{y^{(i)} = j\right\} \log \frac{e^{\theta_j^T x^{(i)}}}{\sum_{l=1}^k e^{ \theta_l^T x^{(i)} }}  \right]
              + \frac{\lambda}{2} \sum_{i=1}^k \sum_{j=0}^n \theta_{ij}^2
\end{align}
$$

...which is similar to the cost function of logistic regression, except that we now sum over the k different possible values of the class label. 

##regularization

We have included in the cost function a regularization component, a weight decay term which penalizes large values of the parameters.

$$
\frac{\lambda}{2} \sum_{i=1}^k \sum_{j=0}^{n} \theta_{ij}^2
$$

With this weight decay term (for any $λ > 0$), the cost function $J(θ)$ is now strictly convex, and is guaranteed to have a unique solution. The Hessian is now invertible, and because $J(θ)$ is convex, algorithms such as gradient descent, L-BFGS, etc. are guaranteed to converge to the global minimum.

##indicator Function

In the equation above, $1\\{\cdot\\}$ is the indicator function, so that 1{a true statement} = 1, and 1{a false statement} = 0. For example, 1{2 + 2 = 4} evaluates to 1; whereas 1{1 + 1 = 5} evaluates to 0.

The following code takes in a vector y with observed labels (from training set) and outputs the indicator function. So for example, for...

$$
\begin{align} 
y &=

\begin{bmatrix} 
3 \\
0 \\
8 \\
\end{bmatrix} \\
\end{align} 
$$

...it ouputs

$$
[0, 0, 0, 1, 0, 0, 0, 0, 0, 0]\\
[1, 0, 0, 0, 0, 0, 0, 0, 0, 0]\\
[0, 0, 0, 0, 0, 0, 0, 0, 1, 0]
$$

An (m,1) label vector is converted into a (m, num_labels) sparse matrix.


```python indicator function
import numpy as np

def groundTruth(y, numLabels):
    m = y.shape[0]
    groundTruth = np.zeros((m, numLabels))
    for row in range(m):
        groundTruth[row, y[row,0]] = 1
    return groundTruth
```

Once we have the indicator function built, the code for the cost function is simple.

```python cost function
import numpy as np

def j(thetas, x, groundTruth, numLabels, regul_lambda):
    thetas = thetas.reshape(numLabels, -1)                    # just making sure well formed
    m = x.shape[0]
    hx = h(thetas, x)                                         # compute hypothesys matrix
    b = groundTruth*(np.log(hx))                              # indicator function computation
    lambdaEffect = (regul_lambda/2)*np.sum(np.sum(thetas**2)) # regularization cost component
    J = -np.sum(np.sum(b))/m + lambdaEffect                   # add up to compute scalar cost
    return J
```
