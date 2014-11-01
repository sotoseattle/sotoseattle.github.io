---
layout: post
title: "MCMC ~ Metropolis Hastings"
date: 2014-02-12 8:01
comments: true
categories: PGN, theory
---

With Gibbs sampling, we changed the assigment of our variables one at a time, and after cycling though all the variables we got to new state, a complete new assignment to all variables. 

For regular chains, Gibbs sampling is a legal MCMC transition, meaning that it results in a Markov Chain with the right stationary distribution. But with high autocorrelations between variables, trying to change the value of one variable at a time tends to leave us in the same state or the same cluster of similar states, because the one variable we change at a time remains strongly constrained by its neighbors. So mixing is slow. 

We’d like to use a transition that lets us make bigger moves around the state space, but we still have to make sure our transition is a legal MCMC. The Metropolis-Hastings algorithm tries to do this. We choose the transition we actually want – one that takes large steps around the state space. Then we modify that transition, using an acceptance probability, that forces it into a legal MCMC transition. 

## Q and A

Our MCMC transition then has two steps. First we choose a proposed new state (x'), using our transition probabilities Q(x → x'). Then we accept that new state, with probability A(x → x'). If the transition is accepted, our new assignment is the proposed new state, but if the transition is rejected, our new assignment is the same as the old one.

For a given proposal transition Q(x → x'), the acceptance probability is given by: 

$$
\begin{align}

A(x \rightarrow x') = min \left[ 1, \frac{\pi(x') Q(x' \rightarrow x)}{\pi(x) Q(x \rightarrow x')} \right]

\end{align}
$$

The key is that A is determined once we know Q, so the art is in finding the right Q. One that:

- is reversible, so if there is a positive probability of getting from x to x', then the probability of getting from x' to x is also positive.

- allows us to go far away places, but without getting so far away that the probability of acceptances becomes too low, because then we won't move at all.

Finding Q is hard, and an art that depends of the PGM at hand. Also, a legal Q is such that for every two states/assignments (x, x'), the probability of transitioning (x → x') is exactly the same as the probability of transitioning in reverse (x' → x), [such property is called 'detailed balance']. 

## Q as Gibbs, or Gibbs as MH

We can insert Gibbs into our MH framework with a trick. We consider that the Gibbs sampling method is our Q(x → x'), our way of finding new assignment (even if in litle steps which violates the MH goal, but it is a trick). Since Gibbs is already legal, all we need is to define an approval distribution A(x → x') that always accepts whatever Q (Gibbs) brings, for example A(x → x') = 1.

## Q as the Uniform distribution

A very simple Q is based on the Uniform distribution. For all the variables in the assignment we randomly choose (independently) new assignments. Brutishly. All at once. Then we compute the transitions by multiplying the specific probabilities of getting that evidence (to and from the initial assignment from/to the new assignment). Now we can compute the acceptance probability A.

The problem is that because we are pulling out assignments out of a hat randomly, most of them will be very unique/unrelated and therefore with low-probability transition. A low probability transition will surely get rejected by the acceptance step, and so we end up lingering in one state rather than exploring the state space. So we may end up with tons of failed trials, tons of staying put, and we only move out of the assignment when we don't go too far.

Ising models and image segmentation applications tend to rely on pairwise Markov networks. In these networks, adjacent variables tend to take on the same values. This makes it hard to explore the space for proposal distributions which change the value of one variable at a time, such as Gibbs or the uniform proposal distribution.