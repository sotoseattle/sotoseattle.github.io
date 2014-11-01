---
layout: post
title: "Kaggle Digit OCR: Softmax Model"
date: 2013-10-06 09:02
comments: true
categories: regression, softmax, kaggle, ocr
---

Instead of using a Random Forest as suggested I am going to start with Softmax regression.

## datasets

The original kaggle/mnist data is therefore divided into e sets:

- a training set with the first 30.000 images (xt, yt) from train.csv.
- an evaluation set of the remaining 12.000 examples (xe, ye) from train.csv.
- the testing set with 28.000 images (x_test) to classify from test.csv.

I also initialize some parameters that I may later optimize:

- lambda, the regularization parameter (L).
- number of labels (LABS = 10), digits 0 to 9.
- number of features (N = 784).
- thetas initialized to zeros as a matrix (10, 784)

```python class KaggleTrain initialization
import math
import pandas as pd
import numpy as np
import scipy as sc
from scipy.optimize import fmin_l_bfgs_b
from sklearn import preprocessing
import matplotlib.pyplot as plt

import sys
sys.path.append( '../../.' ) # logysoft module address
import Logysoft as softmax  # module with softmax utility functions

class KaggleTrain(object):
    def __init__(self):
        # initialize general parameters
        self.L = 7                  # lambda, regularization parameter
        self.LABS = 10              # N. of possible values of Y (labels)
        self.N = 784                # N. of features per image (28x28)

        # load data files
        training_data = pd.read_csv('./data/kaggle/train.csv', header=0)
        testing_data = pd.read_csv('./data/kaggle/test.csv', header=0)
        x = np.array(training_data.ix[:, 1:]).astype('float64')
        y = np.atleast_2d(training_data.ix[:, 0]).T

        # training set
        self.xt = x[0:30000, :]
        self.yt = y[0:30000, :]
        self.gt = soft.groundTruth(self.yt, self.LABS)
        # evaluation set
        self.xe = x[30000:, :]
        self.ye = y[30000:, :]
        self.ge = soft.groundTruth(self.ye, self.LABS)
        # testing set
        self.x_test = np.array(testing_data.ix[:,:]).astype('float64')
```

## choosing lambda ($λ$)

To choose the regularization parameter first we choose the range of lambdas for which to test as [1e-3, 1e-2, 1e-1, 1, 10, 100] and then we zero in in a smaller range (with minor modifications to the code).

For each lambda we find the optimum $θ_{opt}$ in the training set and then compute the cost $J(θ_{opt})$ in the evaluation set. We choose the lambda that works best and minimizes that cost/error measure.

```python choosing lambda
def choose_lambda(self):
    '''train with different regularization parameters and choose
       the one that minimizes the cost in the evaluation set.'''        
    tinit = 0.005* np.random.rand(self.LABS, self.N)

    # initialize some working vars
    rango = np.array([1e-3, 1e-2, 1e-1, 1, 10, 100])
    Jt, Je = np.array([]), np.array([])
    bestC, bestL = 1e+10, 0.0

    # cycle through lambdas and choose the one with lowest cost
    for chosen_lambda in rango:
        t = soft.optimizeThetas(tinit, self.xt, self.gt, \
            numLabels=self.LABS, l=chosen_lambda, visual=True)

        cost_t = soft.j(t, self.xt, self.gt, self.LABS, chosen_lambda)
        cost_e = soft.j(t, self.xe, self.ge, self.LABS, chosen_lambda)

        Jt = np.append(Jt, cost_t)
        Je = np.append(Je, cost_e)

        if cost_e < bestC:
            bestC = cost_e
            bestL = chosen_lambda
    print "\n\nthe best lambda is", bestL

    # plot
    line1 = plt.plot(np.log10(rango), Jt)
    line2 = plt.plot(np.log10(rango), Je)
    plt.setp(line1, linewidth=2.0, label='training', color='b', solid_joinstyle='round')
    plt.setp(line2, linewidth=2.0, label='training', color='r', solid_joinstyle='round')
    plt.xlabel('log10(Lambda)')
    plt.ylabel('J')
    plt.show()
    pass
```

The code not only informs us of the best lambda available but plots the cost curves for each lambda. The blue line represents the cost (error) for each lambda value in the training set. The red line is the cost in the evaluation set and from where we choose $λ$ that minimizes the cost/error. The first chart is log10 of lambda and the second is just lambdalone.


<div style="text-align:center">
{% img /images/kaggle_ocr/choosing_lambda_kaggle_softmax.png 400 500 Training Error (blue) & Validation Error (red) %}
{% img /images/kaggle_ocr/choosing_lambda_kaggle_softmax_2.png 400 500 Training Error (blue) & Validation Error (red) %}
</div>


In our case, we choose a small lambda of 0.002. With input data that is not scaled, the same exercise leads to choosing a $λ$ of around 7.

## optimizing thetas

Once we have the regularization fixed we compute the optimized thetas on the training set and measure the accuracy on the evaluation set.

```python optimize parameters and check accuracy
def check_accuracy(self):
        '''computes thetas on training set, saves them, and checks
           accuracy on evaluation set'''
        tinit = 0.005* np.random.rand(self.LABS, self.N)
        thetas = soft.optimizeThetas(tinit, self.xt, self.gt, self.LABS, self.L)
        thetas = thetas.reshape(self.LABS, -1)
        np.savetxt('./data/kaggle/optimized_thetas.csv', thetas, delimiter=',')

        h = soft.h(thetas, self.xe)
        predictions = h.argmax(axis=1)
        zeros_are_right = np.subtract(self.ye.T, predictions)
        misses = 1.0 * np.count_nonzero(zeros_are_right)
        acc = 1 - misses/len(predictions)
        print 'accuracy:', acc
        pass
```

We achieve an accuracy rate of accuracy: 0.919.

## learning curves

Finally we plot and analyze the learning curves by repeating the exercise for different sample sizes.

The cost (error) on the training set is represent in blue and the cost on the validation set in red. For small training sets the error is small because any curve can be made to fit few data points. The validation error is big because the simplistic model has little to do with reality. 

As we grow the sample size the training error grows (the pains of fitting all points in the curve) while the validation cost diminishes as more complex models generalize better to untrained points.

{% img center /images/kaggle_ocr/learning_kaggle_soft.png 600 500 Training Error (blue) & Validation Error (red) %}

A priori we see that our model does not suffer from a bout of bias. Nevertheless, since we still have room to improve in terms of accuracy we may be suffering from a bit of high variance (over fitting to the training data and lack of generalization). Ways to solve this are:

- increase $λ$. Not really since we know it was optimized and from here we will only rise the error and bias our model.
- gather more training examples. Not available either in kaggle, although we could create new examples by deforming, rotating, etc the existing ones.
- reduce the number of features. I tried with PCA and the results were definitively poorer, maybe because we were indiscriminately removing parameters?
- get creative (more later).

```python learning curves
def learning_curves(self):
    tinit = 0.005* np.random.rand(self.LABS, self.N)
    m, n = self.xt.shape
    sample = np.array([3, 6, 9, 12, 15, 18, 21, 24, 27, 30])*1000
    Jt, Je = np.array([]), np.array([])
    
    for m in sample:
        my_t = soft.optimizeThetas(tinit, self.xt[0:m,:], self.gt[0:m,:], \
            numLabels=self.LABS, l=self.L, visual=False)
        
        Jt = np.append(Jt, soft.j(my_t, self.xt[0:m,:], self.gt[0:m,:], self.LABS, self.L))
        Je = np.append(Je, soft.j(my_t, self.xe, self.ge, self.LABS, self.L))

    # plot (m, Jtr) and (m, Jcv)
    line1 = plt.plot(sample, Jt)
    line2 = plt.plot(sample, Je)
    
    plt.setp(line1, linewidth=2.0, label='training', color='b', solid_joinstyle='round')
    plt.setp(line2, linewidth=2.0, label='training', color='r', solid_joinstyle='round')
    plt.xlabel('Number of Examples')
    plt.ylabel('Cost / Error')
    plt.show()
    pass
```

## Submission

For the submission I optimized parameters for the original set of 42.000 images (not just the 30.000 examples of the training set). 

{% blockquote %}
The accuracy achieved in the Kaggle competition is 92.086%. 
{% endblockquote %}

I also tried a couple of small tweaks:

- Re-scaling \[to range (0,1), mean normalize and feature scale\] and using an optimized $λ$ of 0.002 the kaggle accuracy was a tad smaller at 91.6% 
- Reducing dimensions with PCA while keeping 90 to 99% of variance. The accuracy went down to 90%, proving that PCA may have been removing valuable information. PCA is not always a good recipe for high variance problems.

```python predicting on test set
def test_model_submit(self):
    # compute thetas on whole training set
    tinit = 0.005* np.random.rand(self.LABS, self.N)
    x = np.vstack([self.xt, self.xe])
    y = np.vstack([self.yt, self.ye])
    g = np.vstack([self.gt, self.ge])
    
    # find thetas and save them
    thetas = soft.optimizeThetas(tinit, x, g, self.LABS, self.L)
    thetas = thetas.reshape(self.LABS, -1)
    np.savetxt('./data/kaggle/submit_optimized_thetas.csv', thetas, delimiter=',')
    
    # compute predictions
    m, n = self.x_test.shape
    h = soft.h(thetas, self.x_test)
    predictions = np.zeros((m,2))
    for i in range(m):
        a = h[i,:].argmax()
        predictions[i,:]=[i+1, a]
    print 'To submitt add header: ImageId,Label'
    print predictions[0:10,:]
    np.savetxt('./data/kaggle/predictions.csv', predictions, fmt='%i,%i')
    pass
```

## conclusion

I have read that for classification problems, in practice, different algorithms often yield similar accuracy measures. Most of the time algorithm doesn't matter as much. You can easily reach 90% recognition rate with the simplest algorithm, but every next percent is very difficult to achieve. I seem to have to struck the softmax wall.

