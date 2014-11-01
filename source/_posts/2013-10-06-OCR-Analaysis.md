---
layout: post
title: "Kaggle Digit OCR: Softmax Analysis"
date: 2013-10-06 09:03
comments: true
categories: regression, softmax, kaggle, ocr
---

Reading this [post](http://peekaboo-vision.blogspot.com/2012/12/another-look-at-mnist.html) I realized that we may be able to use a simple logistic regression on a 1-to-1 basis for those cases the softmax model gets wrong. 

We extract the 970 images that we misclassified (out of the 12.000 of our validation set). Then for each image we compute the hypothesis function of each image using the optimized parameters (previously stored). 

Let's get as an example the 42<sup>nd</sup> image from the evaluation set:

{% img center /images/kaggle_ocr/xe_42.png 400 400 Seven %}

The hypothesis function of the image outputs the following probabilities for each label (0 to 9) that we have multiplied by 100: ```[43.79, 7.6e-09, 0.37, 0.0008, 0.12, 0.44, 55.11, 0.02, 0.09, 0.02]```

Our model mis classifies it as a 6 (highest probability of 55%) but it is really meant to be a 0. Now, realize that the second choice, the second maximum is for label 0 (with 43% probability), the right one!. Furthermore, the second choice is at a big distance from everyone else (less than 1%). So even if softmax failed, it was pretty close.

If we could pick and choose the cases where first and second choice were close by, we could tell the model to disregard the softmax choice and instead, for that particular case, perform a logistic regression between the two choices. Generalized to all possible combinations of first and second choices we would need to compute $(n-1)n/2 = 45$ times the 784 parameters.

We can derive from the data that, of the 970 misclassified images on the evaluation set, 608 (or 62%) happen to have as second choice the true class. For example, the most frequent misclassification in this later group happens differentiating between 7 and 9, (69 times out of 608).

Let's make a small experiment and see if it is true that logit works better than softmax when figuring 7s and 9s. We,

- extract from the training and evaluation test all images whose true label is 7 or 9. 
- derive the opt thetas for a simple logistic regression from the extracted training set.
- compare on the extracted evaluation set the accuracy of both, softmax and logistic regression.

Softmax gets it right 90.8% and logit 95.8%. Lets repeat it for the most conflicting pairs previously identified:

|-----------+----------+----------------+-------------|
| pairs     | count    | softmax_acc(%) | logit_acc(%)|
|:---------:|:--------:|:--------------:|:-----------:|
|-----------+----------+----------------+-------------|
|  7 vs. 9  | 69       | 90.8           | 95.8        |
|  4 vs. 9  | 53       | 91.3           | 96.8        |
|  5 vs. 8  | 46       | 86.6           | 96.5        |
|  3 vs. 5  | 40       | 87.6           | 96.0        |
|  1 vs. 8  | 37       | 93.5           | 98.4        |
|  2 vs. 8  | 33       | 89.4           | 97.7        |
|  2 vs. 3  | 32       | 90.2           | 97.3        |
|  3 vs. 8  | 29       | 89.6           | 97.1        |
|  2 vs. 7  | 24       | 91.0           | 98.7        |
|  5 vs. 6  | 21       | 90.2           | 97.8        |
|  0 vs. 6  | 19       | 95.8           | 98.6        |
|  3 vs. 9  | 19       | 90.0           | 98.5        |
|  2 vs. 6  | 18       | 92.9           | 98.5        |
|  0 vs. 5  | 15       | 90.4           | 98.4        |
|  0 vs. 2  | 12       | 93.0           | 98.5        |
|  3 vs. 7  | 12       | 91.2           | 98.4        |
|  5 vs. 9  | 12       | 87.1           | 98.5        |
|  4 vs. 6  | 12       | 94.4           | 99.0        |
|  8 vs. 9  | 10       | 89.2           | 98.7        |
|  4 vs. 5  | 10       | 88.8           | 98.6        |
|-----------+----------+----------------+-------------|
{:.widetable}


<br/>

Here is the module for logistic regression:

```python logit module
import math
import pandas as pd
import numpy as np
import scipy as sc
from scipy.optimize import fmin_l_bfgs_b

# MODULE FOR LOGISTIC REGRESSION
def sigmoid(z):
    return np.vectorize(lambda x: 1/(1+np.exp(-x) + 1E-11))(z)
    #return np.vectorize(lambda x: 1/(1+np.exp(-x)))(z)

def h(t, x):
    '''hypothesis function. probability of input x with params t'''
    return sigmoid(x.dot(t))

def j(t, x, y, lam):
    '''cost function J(theta)'''
    prediction = sigmoid(np.dot(x, t))
    m = x.shape[0]
    J = (-y.T.dot(np.log(prediction)) -        \
            (1-y).T.dot(np.log(1-prediction)) +    \
            (lam/2.0) * np.sum(np.power(t[1::], 2)))/m
    return J

def v(t, x, y, lam):
    '''gradient function, first partial derivation of J(theta)'''
    prediction = sigmoid(np.dot(x, t))
    regu = np.hstack([[0],t[1::,]*lam])
    grad = ((prediction - y).dot(x) + regu)/x.shape[0]
    return grad

def optimizeThetas(tinit, x, y, lam, visual=True):
    '''derive thetas using l_bfgs algorithm'''
    def f(w):
        return j(w, x, y, lam)
    def fprime(w):
        return v(w, x, y, lam)
    [thetas, f, d] = fmin_l_bfgs_b(func=f, x0=tinit, fprime=fprime, maxiter=400)
    if visual:
        print thetas[0:10]
        print f
        print d
    return thetas

def accuracy(t, x, y):
    acc = 0.0
    m= x.shape[0]
    for i in range(m):
        p = h(t, x[i,::])
        if (p>0.5 and y[i]==1) or (p<=0.5 and y[i]==0):
                acc += 1
    return acc/m
```