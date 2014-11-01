---
layout: post
title: "Kaggle Digit OCR: 2 Step Regression"
date: 2013-10-07 09:02
comments: true
categories: regression, softmax, kaggle, ocr
---

Based on the previous posts we are going to extend our softmax model so we classify the handwritten images by using logistic regressions in two steps:

1. First we run our Softmax (multivariate logistic regression) classifier as before.

2. When classifying a particular image, if the first choice has less than 99% probability and the second choice has more than 1%, then we run a logistic regression between those two top choices to make sure we pick the right one. These limits are completely arbitrary, pretty irrelevant and stated to make the concept clearer.

This means computing the 784+1 parameters of the logystic regression 43 times, which is quite hair-raising but not heart-stopping (30 mins in my old mac).

So if it is very clear we go softmax, but if there is a doubt we rely on a more accurate logistic regression between the top two softmax choices. 

{% blockquote %}
The new accuracy from Kaggle is 93.64%. 
{% endblockquote %}

Here is the modified code:

```python kaggle 2 step regression model
import math
import pandas as pd
import numpy as np

import sys
sys.path.append( '../../.' ) # regression modules address
import Logysoft as soft
import Logysterical as logit

class KaggleTop2(object):
    def __init__(self):
        self.LABS = 10              # N. of possible values of Y (labels)
        # load data files
        training_data = pd.read_csv('./data/kaggle/train.csv', header=0)
        testing_data = pd.read_csv('./data/kaggle/test.csv', header=0)
        self.xt = np.array(training_data.ix[:, 1:]).astype('float64')
        self.yt = np.atleast_2d(training_data.ix[:, 0]).T

        # evaluation set
        self.xe = self.xt[30000:, :]
        self.ye = self.yt[30000:, :]
        
        # testing set
        self.x_test = np.array(testing_data.ix[:,:]).astype('float64')

    def optimize_logit_for(self, pair):
        # extract the right images from the set
        [a,b] = [int(pair[0]), int(pair[1])]
        
        data = np.array(pd.read_csv('./data/kaggle/train.csv', header=0)).astype('float64')
        data_t = data[0:, :]
        
        data_a = data_t[data_t[:,0]==a]
        data_b = data_t[data_t[:,0]==b]
        data_ab = np.vstack([data_a, data_b])
        xt = data_ab[:, 1:]
        yt = data_ab[:, 0].astype('int8')
        yt = np.atleast_2d(yt).T
    
        # Perform logistic regression
        xt2 = np.column_stack([np.ones((xt.shape[0],1)), xt])
        yt2 = yt.flatten()
        count = 0
        for i in range(yt2.size):
            if yt2[i]==a:
                yt2[i] = 1
                count +=1
            else:
                yt2[i] = 0    
        
        ini_thetas = 0.005*np.random.rand(xt2.shape[1],1)
        L = 1e+5
        opt_thetas = logit.optimizeThetas(ini_thetas, xt2, yt2, L, visual=False)
        return opt_thetas

    def test_model_submit(self):
        logit_thetas = {}
        
        soft_thetas = np.array(pd.read_csv('./data/kaggle/submit_optimized_thetas.csv', header=None))
        soft_thetas = soft_thetas.reshape(self.LABS, -1)

        m, n = self.x_test.shape
        h = soft.h(soft_thetas, self.x_test)
        predictions = np.zeros((m,2))
        for i in range(m):
            [ml_1, ml_2] = h[i,:].argsort()[-2:][::-1] # 1st and 2nd model choices
            p1,p2 = h[i,:][ml_1], h[i,:][ml_2]
            right_order = True
            if ml_1 > ml_2:
                right_order = False
                s = `ml_2`+`ml_1`
            else:
                s = `ml_1`+`ml_2`
            
            if p1<0.99 and p2>0.01:
                if s not in logit_thetas:
                    logit_thetas[s] = self.optimize_logit_for(s)

                l_t = logit_thetas[s]
                logix = np.hstack([1, self.x_test[i,:]])

                p = logit.h(l_t, logix)
                if (p>0.5):
                    predictions[i,:] = ([i+1, ml_1] if right_order else [i+1, ml_2])
                else:
                    predictions[i,:] = ([i+1, ml_2] if right_order else [i+1, ml_1])
            else:
                predictions[i,:]=[i+1, ml_1]

        print 'To submitt add header: ImageId,Label'
        print predictions[0:10,:]
        np.savetxt('./data/kaggle/predictions_2steps.csv', predictions, fmt='%i,%i')
        pass


k2t = KaggleTop2()
k2t.test_model_submit()
```

## conclusion

We have gained a small improvement of 1.6%. The right direction, yet far away from the results achieved with other models. This seems to suggest a limitation for this problem of a learning function that is linear. In future posts I'll look for models that rely on non-linear functions.

I also wonder what would be the result of using as model logistic regression one-vs-all for each label. And how it compares to plain softmax.


