---
layout: post
title: "Neural Networks Theory: Forwards"
date: 2013-10-17 19:02
comments: true
categories: neural networks, theory
---

No need to introduce how neurons work. The pictures below are self explanatory and better explanations can be founf in the Net. Again thanks to Prof Ng of Coursera fame for making this material so accessible easy so grasp, and to the always great reference of [Stanford Ufldl](http://ufldl.stanford.edu/wiki/index.php/Neural_Networks).

{% img center /images/oct13/neuron.png neuron %}

{% img center /images/oct13/neuron_transmission.gif neuron %}

## digital neurons

Our simplified version has the computation, transformation of input into output, being the logistic regression of the input.

{% img left /images/oct13/one_neuron.png neurotic neuron alone %}

In the picture the +1 input is another input, the $x_0$, also called the 'bias unit' and as we know from logistic regression, always with value one. Sometimes it is explicitly specified, sometimes it is absent from the representations, nevertheless it is always there when we compute things. This single neuron model is a.k.a. Sigmoid (logistic) activation function (where activation function was the g(z)).

$$
h(x) = \frac{1}{ 1  + e^{ -\theta^T x } }
\\
g(z) = \frac{1}{ 1  + e^{-z}}
$$


A neural network is just a bunch of neurons connected together in layers. In the picture below, three neurons in a layer plus another neuron on layer three. Realize that the bias units of layers 1 and 2 are not drawn. The first layer is called 'Input Layer', the last layer is the 'Output Layer', and the ones between are the 'Hidden Layers'.

{% img center /images/oct13/neural_net.png neuron party %}

More terminology: $a_i^{j}$ is the output ('activation') of neuron in layer j. $\Theta^{j}$ is the matrix of parameters ($\theta$) that control the function mapping from layer j to the next one (j+1).

## feed-forward propagation

Let's compute things moving starting at the input, from left to right. The computation of the outputs of layer 2 based on the inputs is:

$$
a_1^{(2)} = g(\theta_{10}^{(1)}x_0 + \theta_{11}^{(1)}x_1 + \theta_{13}^{(1)}x_3) = g(z_1^{2})
\\
a_2^{(2)} = g(\theta_{20}^{(1)}x_0 + \theta_{21}^{(1)}x_1 + \theta_{23}^{(1)}x_3) = g(z_2^{2}
\\
a_3^{(2)} = g(\theta_{30}^{(1)}x_0 + \theta_{31}^{(1)}x_1 + \theta_{33}^{(1)}x_3) = g(z_3^{2}
\\
h_{\Theta}(x) = a_1^{(3)} = g(\theta_{10}^{(2)}a_0^{(2)} + \theta_{11}^{(2)}a_1^{(2)} + \theta_{12}^{(2)}a_2^{(2)} + \theta_{13}^{(2)}a_3^{(2)} ) = g(z_1^{3}
$$

So because we have 3 input units (plus the bias one) and 3 hidden units, the $\Theta^{1}$ would have shape (3,4). Similarly, $\Theta^{2}$ maps from 3 hidden units (plus bias one) to 1 output unit, so the shape is (1,4). Things to consider:

- in the example, $x_i$ are the inputs and work exactly like given $a_{input}$
- as bias units, $a^{1} = x_0 = 1$
- number of params matrices = layers - 1.
- shape of params matrix = out layer size x in layer size [+ bias].

The above equations, vectorized become:

$$
z^{2} = \Theta^{(1)} x = \Theta^{(1)} a^{(1)}
\\
a^{(2)} = g(z^{(2)})
$$

Where x is a vector (4,1), and $z^{(2)}$ is (3,1).

$$
z^{3} = \Theta^{(3)} a^{(2)}
\\
a^{(3)} = h_{\Theta}(x) = g(z^{(3)})
$$

And $z^{(3)}$ is a vector a scalar because $\Theta^{(3)} . a^{(2)}$ is (1,4) x (4,1).

The key to understand is that we are just applying logistic regression to each neuron based on features that are the activation inputs from previous layers. It is obvious in the last unit, and is easy to generalize from it backwards. 

Another take: In the old logistic regression we had some features from which we compute a set of parameters and then predict the hypothesis function. Now, we can create a new set of features based on the initial features, and use these new features to compute the logistic regression prediction. Is just applying the same logit in a chained manner. The new features will be a set of different, derived, more nuances and more complex features than the initial ones. 

The more to the right the layer, the more complex the features because of more manipulation. It is very illustrative the example of Prof. Ng replicating the function XNOR with just three neurons to appreciate how it is possible to create very complex features by applying simple functions to each unit, so they compound the complexity along the way.

{% img center /images/oct13/x1xnorx2.png compounded complexity %}


The code couldn't be simpler.

```python feed forward
def sigmoid(z):
    return np.vectorize(lambda x: 1/(1+np.exp(-x) + 1E-11))(z)

def sigmoid_gradient(z):
    return (sigmoid(z)*(1-sigmoid(z)))

def feedforward(theta_matrix, input_v):
    '''one layer forward computation, assumes input_v has no bias'''
    m = input_v.shape[0]
    input_v = np.hstack([np.ones((m,1)),input_v])
    z = input_v.dot(theta_matrix.T)
    return sigmoid(z)

def h(thetas_rolled, layers, one_x):
    thetas = unroll_thetas(len(one_x), layers, thetas_rolled)
    a = one_x
    for t in thetas:
        a = sigmoid(np.hstack([1,a]) .dot(t.T))
    return a
```




