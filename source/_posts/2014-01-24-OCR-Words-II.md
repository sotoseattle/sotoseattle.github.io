---
layout: post
title: "OCR Words (II)"
date: 2014-01-24 08:01
comments: true
categories: PGN, theory
---

Now that we have an inference engine and we have added a way to find the MAP we can continue with our OCR example [OCR Example](/blog/2013/11/16/OCR-Words). But first of all we are going to add a new factor to the mix.

For the given word we can compute how similar the images are between each two. So if two images (handwritten letters) look very much alike, and we are pretty sure that the first one is an "a", then we want to load the scale for the second in favor of being an "a" too.


{% img center /images/jan14/ocr_similarity_factor.png 500 Similarity Factors%}

If we compute this factor for all two different letters in a word we end up with way too many connections among variables, making the net an impossible endeavour (more than 5 letters break havoc). For this reason we will only select the two top similarities. In the example we assume that the top two similar letters are c ~ a, and r ~ p.

The exact mathematical details are not important and have being adapted from the example I am following from Prof. Daphne Koller, Stanford University. Suffice to say that for the selected variables we build a 2 var factor with all values equal to 1 except for the diagonal, with values equal to the similarity score between the images. This way, the higher the score (the more similar the images), the bigger boost to the probability of both variables refering to the same letter.

```python Top 2 Similarity Factors
def similarity(img1, img2):
    meanSim = 0.283 # Avg sim score computed over held-out data. A tweak to scale and boost accuracy.
    cosDist = np.dot(img1.T, img2) / (np.linalg.norm(img1) * np.linalg.norm(img2))
    diff = (cosDist - meanSim) ** 2;
    if (cosDist > meanSim):
        sim = 1 + 5*diff;
    else:
        sim = 1 / (1 + 5*diff);
    return sim

def image_simil_factor(var1, var2, sim):
    sol = np.ones(k) + np.diag(np.array([sim-1]*k))
    f = Factor.Factor([var1, var2])
    f.values = sol
    return f
```


