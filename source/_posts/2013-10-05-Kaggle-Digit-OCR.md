---
layout: post
title: "Kaggle Digit OCR: The Data"
date: 2013-10-05 09:02
comments: true
categories: regression, softmax, kaggle, ocr
---

My first ML program is going to be the [Kaggle OCR digit competition](http://www.kaggle.com/c/digit-recognizer). 

The original data comes in two files: train.csv and a test.csv. Each file contain gray-scale images of hand-drawn digits, from zero through nine.

Each image is 28 pixels in height and 28 pixels in width, for a total of 784 pixels in total. Each pixel has a single pixel-value associated with it, indicating the lightness or darkness of that pixel, with higher numbers meaning darker. This pixel-value is an integer between 0 and 255, inclusive.

The training data set, (train.csv), has 785 columns and 42.0001 rows. The first column, called "label", is the digit that was drawn by the user. The rest of the columns contain the pixel-values of the associated image. The first row is a header and each subsequent row holds an image in vectorized form for a total of 42.000 images

The test file is similar but without a label column, which is the goal of the competition.

We can visualize each handwritten digit with the following code:

```python visualize handwritten digit
x = numpy.array(...)	# all the training images
y = numpy.array(...)	# all the training labels

def visualize(example_number):
	print 'should be a', y[example_number]
	matplotlib.pyplot.gray()
	matplotlib.pyplot.imshow(x[example_number,:].reshape(28,28))
	matplotlib.pyplot.show()
```

For example, from train.csv, the 6<sup>th</sup> row is: [7, 0, 0, ..., 82, 152, 71, 51, 51, ..., 0, 0, 0, 0, 0.]

Where the first digit tells us that the "real" number is a "7" and the following 784 numbers can be reshaped into a grid to show the following image:

{% img center /images/kaggle_ocr/mnist_good_7.png 400 400 Seven %}

Not everything is so clear cut. Two additional examples from the training set: a One and a Seven, which could be misclassified even by hand.

<div style="text-align:center">
{% img /images/kaggle_ocr/mnist_1.png 400 400 One %}
{% img /images/kaggle_ocr/mnist_7.png 400 400 Seven %}
</div>