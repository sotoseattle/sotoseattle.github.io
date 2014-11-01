---
layout: post
title: "Kaggle Digit OCR: SVM (top 5)"
date: 2013-10-13 18:02
comments: true
categories: svm, kaggle, ocr
---

SVM is much easier to implement than our previous regressions because it is already been coded in the package [scikit learn](http://scikit-learn.org/stable/modules/svm.html) (sklearn). All that is needed is to tweak the hyperparameters to achieve a high accuracy.

My data set is divided as before, 30.000 images for training and 12.000 for evaluation. SVM works best with scaled input so my Xs have been pre-processed so all pixel values are in the range {0,1}, and each feature has zero mean and unit variance. For this we also use sklearn preprocessing utilities.

I have tried two non-linear approaches gaussian and 4<sup>th</sup> degreee polynomial, to see how they compare to the most rudimentary linear approach of past runs.

## gaussian kernel

For the Kaggle digit recognition competition I have employed a Gaussian kernel (RBF) with a sigma equal to 0.001. The safety margin constant C is set to 14. Both parameters were found after a few trail and error runs. SVM is pretty slow so the trick is to use very small training/eval sets (2000 images) to zero in the coeficients. Later on we can better the result with the whole sets.

```python svm rbf on test data
m = x_test.shape[0]
clf = svm.SVC(C=14, kernel='rbf', gamma=0.001, cache_size=200)
clf.fit(x[0:42000,:], y[0:42000])
predictions = clf.predict(x_test).values
```

{% blockquote %}
The accuracy on the evaluation set is 96.55% and on the test set (Kaggle) is 96,91%. 
{% endblockquote %}

The result beat the previous 2-regressions approach (93.6%) by a high margin.

## polynomial kernel

4<sup>th</sup> degreee : $(\gamma \langle x, x'\rangle + r)^4$ with a hyperparameter r of 0.38, and a regularization constant C of 8.3. The hyperparameters were found using the StratifiedKFold and GridSearchCV from the scikit package from which we can also produce a heatmap.

{% img /images/kaggle_ocr/choose_params_svm.png searching for hyperparams %}

```python svm call
clf = svm.SVC(C=6.2, kernel='poly', degree=4, coef0=0.48, cache_size=200)
```

The accuracy on the evaluation set is 97.9%. 

## twerking the training data

But I am not done. I want to try a crazy idea I just found, Virtual SVM! 

Kudos to [Jet Goodson](http://www.jetgoodson.com/blogged.php?ident=49) and his clever way to put this Virtual SVM to work. It is based on the paper [Training Invariant Support Vector Machines](http://www.cs.berkeley.edu/~malik/cs294/decoste-scholkopf.pdf) by Dennis DeCoste and Bernhard Scholkopf.

The trick is that when you run an SVM you get as a byproduct a set of Support Vectors, which are nothing but the subset of the training examples that the SVM actually bases its decisions on. Those are the training examples that define the hyperplanes.

`clf.support_vectors_` gives us a matrix of support vectors, which in our case are around 10,000. And `clf.support_` gives an array with the index of the training examples that are the support vectors. We can check this:

```python python console
clf.support_
# array([    5,    69,   146, ..., 41962, 41992, 41999], dtype=int32)
assert np.allclose(xt[5,:], clf.support_vectors_[0,:])
```

So if we want to artificially create new examples from the old ones (moving, blurring, etc.), we should only modify the ones that count. In order to modify the image, since I am a newbie, I'll just move the image to each side [N,S,W,E] a tiny bit (one pixel), and rotate 10 degrees clockwise and counter-clockwise. 

For every image of the support vectors set I can get an additional 6 twerked images. This totals a new training set of around 10.000 x 7 = 70.000 images. This is a stretch for a  SVM but nothing compared that to 42.000 x 7 = 294.000 of directly applying the transformations to the training set.

{% blockquote %}
The accuracy on the evaluation set is 98.225% and on the test set (Kaggle): 98.086%. 
{% endblockquote %}

[Important resource](http://scipy-lectures.github.io/advanced/image_processing/index.html) to start understanding how to manipulate images in python.

```python Twerk SVM for test set
def twerk(x_old_set, y_old_set):
    m,n = x_old_set.shape
    x_new_set = np.zeros((7*m, n))
    y_new_set = np.zeros((7*m,))
    j = 0
    for i in range(m):
        # add the original one
        x_new_set[j] = x_old_set[i,:]
        y_new_set[j] = y_old_set[i]
        
        x = x_old_set[i,:].reshape(28,28)
        y = y_old_set[i]
        
        # move north 1 px
        j +=1
        x_new_set[j] = np.vstack([x[1:,:], x[27:,:]]).flatten()
        #x_new_set[j] = np.roll(x, 2*k[0], k[1]).flatten()
        y_new_set[j] = y
        # move south 1 px
        j +=1
        x_new_set[j] = np.vstack([x[0:1,:], x[0:27,:]]).flatten()
        y_new_set[j] = y
        # move left 1 px
        j +=1
        x_new_set[j] = np.hstack([x[:,1:], x[:,0:1]]).flatten()
        y_new_set[j] = y
        # move right 1 px
        j +=1
        x_new_set[j] = np.hstack([x[:,27:], x[:,0:27]]).flatten()
        y_new_set[j] = y
        # rotate 10 degrees
        j +=1
        x_new_set[j] = ndimage.rotate(x, 10, reshape=False).flatten()
        y_new_set[j] = y
        # rotate -10 degrees
        j +=1
        x_new_set[j] = ndimage.rotate(x, -10, reshape=False).flatten()
        y_new_set[j] = y
            
        j += 1
    return [x_new_set, y_new_set]

def prep_submit():
    clf = svm.SVC(C=6.2, kernel='poly', degree=4, coef0=0.48, cache_size=200)
    clf.fit(x[0:42000,:], y[0:42000])
    
    x_support = clf.support_vectors_
    y_support = np.array(y[clf.support_])
    [x_new, y_new]= twerk(x_support, y_support)
    clf.fit(x_new, y_new)
    
    a = clf.predict(x_test)
    b = np.arange(1,28001,1)
    predictions = np.vstack([b, a]).T
    print 'To submitt add header: ImageId,Label'
    np.savetxt('./data/kaggle/predictions_svm.csv', predictions, fmt='%i,%i')
```

## last experiment

For SVM the size of the training set makes a difference. I am going to try to run it all on the 70,000 images, the training set plus the testing set with the added predictions from the best run. 

This is totally endogamy and an article of faith no-no but I figured:

- since I am already geting 98% right, if at least I can reinforce the good ones, the better performance from that reinforcement of the good ones may be enough to counteract the added errors from that 2% wrongly labeled.
- it is great way to confirm the article of faith.

With twerking et all, we do much worse, the new Kaggle rating is 97.95%

## no, seriously, the last one

I thought: why 28x28 and not bigger? Or smaller? The difference is the amount of padding around the image. I have read that the samples are already pretty centered and aligned, so why not give more importance to the center of the picture and less as we go out? What would happen if we initially crop the image from 28x28 to 20x20? It seems that those 4 pixel bands are pretty much empty. Maybe focusing in the inner image we can better train the model.

```python croping the images
m = x.shape[0]
new_x = np.zeros((m,400))
for i in range(m):
    old = x[i,:].reshape(28,28)
    new_x[i,:] = old[4:24,4:24].flatten()
x = new_x
```

Doing the same as before (re-optimizing hyper params => {C= 2.6, r= 0.34}, twerking as before, etc) we get:

{% blockquote %}
Accuracy on evaluation set: 98.7%
Score on the test set (Kaggle): 98.87%. 
{% endblockquote %}

It is a huge jump of almost a percentage point only based on munging the input!

The new position is 80 out of 1956 participants (as of today), in the top 5%! Consider that my first trial a few days ago (a simple linear softmax model) placed me at position 1,654, in the bottom 15%.

## conclusion

Again, once reached the plateau of the model it is very difficult to raise the bar.

The key is to be creative and look for out-of-the-box approaches.

The single most important aspect is to know the data, visualize in different ways to understand it and to analyze correctly where the model fails.
Finally, I suspect that, with a bit of further image manipulation, the accuracy can be raised a bit more. I'll leave this to a future competition.
