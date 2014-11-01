---
layout: post
title: "Neurotic Titanic"
date: 2013-10-22 21:02
comments: true
categories: kaggle, svm, neural networks, titanic
---

I have spent the last 13th submissions to beat my previous position (77<sup>th</sup> with 81.81% accuracy) without result. I tried concocting and adding new features, combining them in weird ways, estimating the missing values to the highest accuracy possible. All to no avail.

I have tried not only screwing with the SVM, but Neural Networks with the code of previous posts. The Neural Nets work great but tend to overfit tremendously. I get 84-88% in CV only to get 75-80% on the Kaggle test set. The overfitting got me thinking about ways to simplify so the model could generalize better. This is what I have learn:


## less features are better than more

From my previous like 10 features I have realized that we get the same results with less. Actually, the title level alone will give you 79% accuracy on the cross validation set. Then you can experiment adding new features to see how well you do and realize that in order of incremental power come: family (as Parch + SibSp), Fare paid, Pclass, overseer, charisma (binAge * Pclass), Port_Q (embarked on Queenssomething), ....

After the first 5 you don't add much more and a bit further you star to add noise and lose generalization power. This means that things like Age, Sex or Embarked port are not even considered!


## simpler beats complex (sometimes)

I tried to simplify as much as possible the features. First I only use 5 ('title_level', 'family', 'Fare', 'Pclass', 'overseer'). Then I tried to simplify them:

I started estimating the correct Age (as bin) with a SVM and then with a Neural Network. The SVM got 68% right on CV, and the NN north of 90%. The problem is that Age is not key, the title already incorporates all the age information that is relevant. So out it went.

The same with Sex. Title already incorporates the info. Adding Sex, in this case, only confuses things.

On title I aggregate it to just ['mr', 'mrs', 'mss', 'master', 'rev', 'major', 'sir', 'col', 'capt', 'the countess']. The less the merrier.

Family went from a sum of relatives to 3 classes, [traveling alone, 1 relative, more than 1 relative on board].

Overseer, was a bit more built up. essentially for each passenger I look for other relatives and companions that survived. For women, if the survivor was older give 10 points, if not get just 1. For men, if the survivor was a man older by 20 years give 10 points, if not get 1. Any other way get -1. The idea already explained that in a crisis we gather with who we know, and that the elder look after the younger (up to a point).

By the way, I have been trying avoid 0 values, in the suspicion (material or not) that zeros are traps from which is difficult to escape (when most we do is multiply stuff).

## SVM or NN => why not both?

With the above simplifications either SVM or NN got stuck under 80%. There was no way to move the dial so I tried a crazy approach: run them both (with optimized hyperparameters) and then:

- commit when both agree,
- when they disagree on the outcome, choose the one with the highest assigned probability (yes, SVM can estimate the probability too!)

Stupidly simple, and with great results. With my latest iteration, number 26, I landed on the 57 spot of 7,035 participants (as of now).

{% blockquote %}
Score on the test set (Kaggle): 82.775%. 
{% endblockquote %}

The NN alone (submission 27) is a big network with three hidden layers and overfits the data to death (Kaggle 80.38%). The SVM alone does not get past 80% either. But both of them working together work wonders. I am sure more could be extracted from this approach, but I need to get going and I should try a new competition and learn a new model.
