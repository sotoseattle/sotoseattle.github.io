---
layout: post
title: "Data Science London"
date: 2013-10-28 21:02
comments: true
categories: kaggle, svm, neural networks, titanic
---

This is a post about my third tutorial competition, Data Science London.

The training data consists of 1,000 records with 40 features. We need to classify each record as 1 or 0. The test set consists of 9,000 additional records.

After playing a bit with the training set one realizes that there is a ton of noise in the data, and that we need to carefully distill the features to get something of value. Furthermore, the noisy data is supported by the fact that the features were created artificially for the purpose of this tutorial competition.

## feature selection

To start cleaning up the data a first approach is to see which features give the most predictive power. To do this I use an SVM with Gaussian kernel. Then I find the feature that gives the most predictive power. Then I repeat the process finding the feature that added to the previously selected achieves the highest accuracy. 

Another way is to use the ExtraTreesClassifier from the scikit package which results in the following chart of the importance of features ([link](http://scikit-learn.org/stable/auto_examples/ensemble/plot_forest_importances.html#example-ensemble-plot-forest-importances-py)):

{% img center /images/kaggle_scilondon/features_power_svm.png 600 %}

|--------------|-----|----|----|---|----|----|---|----|----|----|----|----|----|----|----|----|----|
| ||||||||||||||||||
|:-------------|----:|---:|---:|--:|---:|---:|--:|---:|---:|---:|---:|---:|---:|---:|---:|---:|---:|
|--------------|-----|----|----|---|----|----|---|----|----|----|----|----|----|----|----|----|----|
|SVM          | 14  | 12 | 29 |  6| 36 | 39 | 7 |  4 | 28 | 19 | 22 | 18 | 38 |  3 | 32 | 34 | 13 |
|TreesClassif | 14  | 12 | 6  | 39| 36 | 18 | 4 | 32 | 38 | 29 | 28 | 23 | 22 | 34 |  7 |  9 | 10 |
|--------------|-----|----|----|---|----|----|---|----|----|----|----|----|----|----|----|----|----|
{:.widetable}


<br/>

Only using the feature number 14 we already guess right more than 2/3 of the times. As we add features we increase the accuracy until it stabilizes somewhere between the first 10 and 22 features. Then, the added noise starts to degrade the model and lose predictive power. Optimizing hyper-parameters for the first 18 features we achieve 93.6% on cross validation and 91.84% on the 9,000 samples test set.

ALTERNATIVE WAYS TO CHOOSE FEATURES FROM SCIKIT

## dimensionality reduction

Even after choosing the set of features we still have way too much noise in the data. One way to 'clean it up' is through dimensionality reduction. This has a good and a bad aspect. On one hand, it is a dangerous practice since what we do is project into a lower dimension space all the features and in the process we loose data, maybe valuable. On the other hand, it allows us to apply 'whitening', removing data points that are close by and highly correlated.

In this case it pays to apply PCA because I rather start with less valuable data but more salient, than more noisy and diffuse information. Here are two images with accuracy on the first 100 records of training an SVM on the subsequent 900 for different sets of features and dimensions. The left one comes from the main features derived from the tree classifier, the right most from the cycling svm:

{% img left /images/kaggle_scilondon/feat_dim_trees.png 420 Features from trees %}

{% img right /images/kaggle_scilondon/feat_dim_svm.png 420 Features from SVM%}

<br/>


My SVM set achieves a maximum accuracy of 97% for the first 14 features reduced to 12 dimensions. The forest one achieves a top 96% for a wider set of combinations (> 12 features and 12-13 dimensions).

As an aside, I tried all kinds of different approaches available in the scikit package for dim reduction, except for the 'kernel pca' that took too long. The best result was achieved with Random PCA on my reduced set of 14 features and reduced to 12 dimensions. These 12 dimensions explain something like half of the variance when considering the whole feature set, and around 90% of the variance in the reduced feature set.

{% img center /images/kaggle_scilondon/explained_var.png 500 Variance explained from dimensions%}

One final note. It is easy to code the PCA algorithm. Nevertheless I used the SciKit tool because it includes the Random PCA and whitening (which still escapes me). Here is the code for bare-bones PCA:

```python barebones PCA
def pca(X):
	m, n = X.shape
	U = np.zeros(n)
	S = np.zeros(n)
	Sigma = np.cov(X.T)
	print 'Sigma shape should be nxn:', Sigma.shape
	print 'U will be m x ', np.min(Sigma.shape)
	U, S, V = np.linalg.svd(Sigma, full_matrices=False)
	return [U, S]

def shrink(example_norm, U, dimensions):
	Ureduce = U[:, 0:dimensions]
	Z = example_norm.dot(Ureduce)
	return Z

def inflate(shrunk_data, U, dimensions):
	Ureduce = U[:, 0:dimensions]
	X_rec = shrunk_data.dot(Ureduce.T)
	return X_rec
```

## engorging the training by guessing

We have an inherent asymmetry: 1000 training records and 9000 for testing. If we could pass records from the testing to the training set we could boost the model's power. The problem is that we don't have the labels of the testing set records. But they are so many that we could run our model on the testing set and believe that the few predictions made with very high probability are so certain as to take them as 'real'. 

In an initial pass of the SVM over the testing set we select the predicted records with probability higher than 99% and add them to our training set. We measure the accuracy on a separated cross validation set and store the result. We repeat the process over and over and only stop when the accuracy on the CV set goes down. We end up with around 8,508 records, 2,316 support vectors, and an accuracy of around 97% on cross validation.

The intuition behind the idea is that although we don't gain anything from certain added records (because we were already pretty sure where they fell), the addition to the training set will allow other less apparent features and subtleties gain contrast and predictive power.

At the end of this process we have a huge, if slightly screwed up training set that gives us the following accuracies against our cross validation set:

|------------|-----------|--------|----------|---------|
|	         | precision | recall | f1-score | support |
|:-----------|:---------:|:------:|:--------:|--------:|
|------------|-----------|--------|----------|---------|
| label 0    |0.97       |0.97    |0.97      | 149     |
| label 1    |0.97       |0.97    |0.97      | 151     |
|------------|-----------|--------|----------|---------|
| avg / total|0.97       |0.97    |0.97      |300      |
|------------|-----------|--------|----------|---------|
{:.widetable}

<br/>

## experiment with NN

As an experiment, we train a neural network with quite an excess of nodes [input: 18, hidden\_1: 36, hidden\_2: 36, output: 2] on the engorged training set. Then for each prediction made by SVM and NN, in case the models disagree, whoever assigns the highest probability to his choice decides.

Run on the CV set we got the same result:

|------------|-----------|--------|----------|---------|
|	         | precision | recall | f1-score | support |
|:-----------|:---------:|:------:|:--------:|--------:|
|------------|-----------|--------|----------|---------|
| label 0    |0.97       |0.97    |0.97      | 149     |
| label 1    |0.97       |0.97    |0.97      | 151     |
|------------|-----------|--------|----------|---------|
| avg / total|0.97       |0.97    |0.97      |300      |
|------------|-----------|--------|----------|---------|
{:.widetable}

<br/>

Both models differed on just 10 samples (3%) and in 5 cases the NN decided. The NN didn't do better than the SVM, it just had a different opinion on some borderline cases.


## final submission

I have rerun my algorithm multiple times trying to find the best initial feature set and best randomized path that engorges the data. My last few consistent scores were between 95.45 and 95.827.

My final submissions came down to: 

- a newly distilled feature set of 20 features, 
- tweaked hyper-parameters,
- the patience to run the engorgement algorithm until I got 97.66% on CV
- 8,808 training records guessed with p>0.99 and 2,635 support vectors
- and using the mix model

which achieves a 95.566% (a tad better than the SVM alone). 

Two days back, out of chance with the same mix strategy on a randomly optimized set I got my best (though I suspect overfit) result:

{% blockquote %}
Kaggle's position 9 with 95.827% accuracy.
{% endblockquote %}


