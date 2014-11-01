---
layout: post
title: "Softmax Theory:  $∇(θ)$"
date: 2013-10-03 09:02
comments: true
categories: regression, softmax, theory
---


There is no known closed-form way to solve for the minimum of $J(θ)$, and thus as usual we'll resort to an iterative optimization algorithm such as gradient descent or L-BFGS. Taking derivatives, one can show that the gradient is:


$$
\begin{align}
\nabla_{\theta_j} J(\theta) = - \frac{1}{m} \sum_{i=1}^{m}{ \left[ x^{(i)} ( 1\{ y^{(i)} = j\}  - p(y^{(i)} = j | x^{(i)}; \theta) ) \right]  } + \lambda \theta_j
\end{align}
$$

By minimizing $J(θ)$ with respect to $θ$, we will have a working implementation of softmax regression. Note that the gradient output is flatten from a matrix form to a vector form. This is needed format for the gradient descent algorithm to work.

```python gradient function
import numpy as np

def v(thetas, x, groundTruth, numLabels, regul_lambda):
    thetas = thetas.reshape(numLabels, -1)
    hx = h(thetas, x)
    m = x.shape[0]
    grad = ((groundTruth-hx).T.dot(x))/(-m) + (regul_lambda*thetas)
    grad = grad.flatten(0)
    return grad

```

##checking the gradient computation

It is very a healthy habit to check gradients numerically before proceeding to train the model. The norm of the difference between the numerical gradient and your analytical gradient should be small, on the order of 10 − 9. The following code uses some random data to compare the above gradient against a numerical approximation based on the formula


The key to compute the gradient by hand is to keep all $θ$ fixed while we change a single one by a small amount (epsilon) $\theta_j := \theta_j - \alpha \nabla_{\theta_j} J(\theta) (for each j=1,\ldots,k)$.


```python checking gradient with numerical approximation
import numpy as np

def grad_by_hand(thetas, x, groundTruth, numLabels, regul_lambda):
    epsilon = 1e-4
    t = thetas.flatten(0)
    n = t.size
    grad = np.ones((n,)).astype('float64')
    for i in range(n):
        t1 = np.copy(t)
        t2 = np.copy(t)
        t1[i] += epsilon
        t2[i] -= epsilon
        a = j(t1, x, groundTruth, numLabels, regul_lambda)
        b = j(t2, x, groundTruth, numLabels, regul_lambda)
        grad[i] = (a-b)/(2*epsilon)
    return grad

x_check = np.random.rand(10,10)
y_check = np.random.randint(1,11, (10,1))
g_check = soft.groundTruth(y_check, numLabels)
t_check = np.random.rand(10,10).astype('float64')
g_theo = v(t_check, x_check, g_check, 10, l)
g_hand = grad_by_hand(t_check, x_check, g_check, 10, l)
diff = np.linalg.norm(g_hand - g_theo) / np.linalg.norm(g_hand + g_theo)
assert diff <= 1e-9
```












