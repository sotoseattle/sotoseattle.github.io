---
layout: post
title: "MCMC Gibbs - Theory"
date: 2014-02-9 08:01
comments: true
categories: PGN, theory
---

In the previous three posts we saw that we can compute approximate marginals in the mini Ising grid with uneven results. The more polarized the potentials, the more contradictions in the messages passed, the more difficult to converge to the true marginals.

In the following posts we are going to approach this problem (computing the aproximate marginals) from a different perspective, the Monte Carlo way. Beware that my grasp of these matters is flimsy at best, so excuse any mistakes I'll made.

## Sampling for Inference

Our PGM is based on, and is a reflection of, a joint distribution of all its variables and the independence relationships among them. And we have already seen that if we have this joint distribution we have all the information necessary to answer any questions. Now, we also saw that except for the simplest graphs it is impractical to compute this joint distribution due to its size. 

Imagine if we could come up with a way to sample this unknown and huge distribution. A way to generate perfectly viable, realistic assignments to all the variables in the joint distribution. With a sample big enough, we can estimate the answers we need, and we could get the approximate marginals.

For a Bayes Network, independently of its size this is doable. We start at the root nodes (eldest parents), roll a dice to see which assignments we get to those variables, and once computed start walking down the graph rolling dice based on the reduced tables of each node. When we have assigned values to all variables we have a complete assignment to the joint distribution. With enough of these we can infere estimated parameters. 

For a Markov Network the previous process does not work, because there are no parents, no direction of flow. It is just a bunch of nodes talking to each other without concert and whose potentials depend on each other's assignments. An egg and chicken conundrum.

## Markov Chain

Markov Chain Monte Carlo (MCMC) gives us a way to generate samples from the joint distribution behind a Markov net using a trick over Markov Chains. (chains and nets, even by the same guy, are unrelated concepts).

A Markov chain consists of a set of states, where wach state has transition probabilities specifying the probabilities of stepping to each other state from that state. We start at some initial state, and in each step we move to another state, obeying the transition probabilities. Now, _some_ Markov chains have the property that if we run them (hopping around) indefinitely, it will always converge to the same stationary probability distribution $ \pi $ for the states: that is, if we run it forever, the probability of being in some state s will be the same p no matter what the initial state was. So it won't matter where you start, or where you went, because in the end, you'll know where you'll end up exactly (probabilisticly speaking, that is).

Better explanation, tons of examples and good writting in Wikipedia. Highly informative.

{% img center /images/feb14/markov_example.png 500 Example of Markov Chain%}


## Mutating our PGM into a Markov Chain

Back to our networks and graphs. We are going to build a Markov Chain where the states from the Markov chain will correspond to complete assignments to the variables of the model. In this way the probability to pass from one state to the next one corresponds to the probability of passing from one assignment of variables to another assignment. And if the whole mess converges over time, we should reach a set of constant probabilities for each assignment possible. Alas, the joint distribution. And sampling from it would get us to a good approximations of the marginals.

The trick is to come up with the right transition probabilities (between complete assignments / states) so a) the chain converges and 2) to the joint distribution we are after. There are different methods to get these transition probabilities right. We start with the simples one, Gibbs.

For example, given a Markov network of only three nodes, a clique. Each node holds a single binary variable [0,1,2]. Each node has a singleton factor on its variable, respectively var 0: [0.4, 0.6], var 1: [0.4, 0.6] and var 2: [0.6, 0.4]. There are additional pairwise factor between each pair of variables (0,1), (0,2) and (1,2), all with the same potentials [agree with value 1.0, and disagree with value 0.2]. The representation of such PGM is very simple, three nodes fully connected.

{% img center /images/feb14/MN_simple.png 300 Mini Markov Net%}


Since all variables are binary, the total possible assignments is 2^3 = 8. We construct a Markov Chain with 8 nodes, one for each assignment. The connections between these nodes, better called transitions since they are directed edges, represent the ability to pass from one assignment to another. So if I am at assignment (0,0,0) and the next assignment I get is (0,0,1), then there is a transition between them.

Using the Gibbs transition process (to be studied below), we can collect a lot of samples from the distribution, a lot of assignments. The order of the samples collected will tell us about the individual hops, from and to which assignments me move in a single step, giving us the transitions (connections between nodes), and just counting the times we transition between each pair of nodes we can derive the transition probabilities.

In the following figure I have drawn the 8 nodes of the derived Markov Chain, and only drawn the transitions of two nodes, assignments (0,0,0), and (0,0,1) for clarity. We can see that once in assignment (0,0,0) there is a high probability of staying put (86.8%), while the probability of remaining in (0,0,1) is only 1%. Also we see that the highest probability for moving from (0,0,1) is to transition to (1,1,1) with 55.8% probability. Beware that these transition probabilities are not the stationary probabilities, but used to estimate them (furthermore there are only 8 stationary probabilities, because they are the probabilities of being at that assignment).

{% img center /images/feb14/MC_simple.png 400 Mini Markov Chain%}

This is an exercise to visualize how the MC from a PGM looks like. For normal PGM it is just too messy and wild, and trying to visualize is futile.

## Gibbs

We start at some randomly chosen initial state, a  randomly chosen initial assignmentto all variables in the Markov network. 

To move to the next state, we iterate through the variables. We replace each value in turn with a newly-selected value using probabilities _conditional on the values of all the other variables_ – the new values for the variables we already changed this iteration and the old values for the variables we haven’t reached yet. When we have gone through all the variables, the new values, the new assignment to all variables, becomes our new state, and we’ve made one step in the Markov chain. 

Now for some esoteric theory. A Markov chain is regular if there exists an integer k such that, for every x, x’, the probability of getting from x to x’ in exactly k steps is > 0. The key is that first we choose the k, then that should hold for every pair of states! 

From that definition we can extract the following: an MC is regular if the following sufficient conditions are met:

*  Every two states are connected (I can go from every state to any other state, with positive probability)

*  For every state, there is a self-transition (there should always be a positive probability of satying put in the state).

The importance of being regular is that by article of faith a regular Markov Chain converges to a unique stationary distribution regardless of start state. Aha!

Our Gibbs way of sampling creates an MC (derived itself from the PGM) that is regular, and therefore it has the same stationary distribution as the distribution of our PGM. So, in the long run, I can find/collect new states/assignments that show up with probability equal to the joint distribution of the PGM. 

Finally, as another aside, for a Markov Network, another sufficient condition is that if all factors are positive, the Gibbs chain is regular. Beware, that being regular does not mean it is efficient and mixes.



