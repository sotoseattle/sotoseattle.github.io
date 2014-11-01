---
layout: post
title: "MCMC Gibbs - Code"
date: 2014-02-10 08:01
comments: true
categories: PGN, theory
---

## How to sample a distribution

The first method we need is to find a random assignment given a discrete distribution. For example, if we have a distribution over a single variable with cardinality 3, [p0, p1, p2] = [0.25, 0.35, 0.4] we:

* create the cumulative distr => [c1, c2, c3] = [0.25, 0.60, 1.0]
* pick up a random number r between 0 and 1 (based on uniform dist), let's say we get 0.43
* create a new array based on [c >= r] => [False, True, True]
* the first True gives us the assignment, in this case 1 out of possible assignments (0, 1, 2).

```python Sample Randomly From Given Distribution
def randSampleDist(self, card, probs):
    cum_prob = np.cumsum(probs)
    u = uniform(0., max(cum_prob))
    included = cum_prob>=u
    sample = np.where(included==True)[0][0] # get first
    return sample
```

## Find distribution for a variable given a complete assignment

Now we are going to get the distribution of a variable given that all other variables have been assigned. We operate in log space for easier computations.

The key is to understand that for each factor fu that includes the variable we can reduce it with the assignment (excluding the sampling variable) and marginalize for all variables (excluding the sampling variable) to come up with the distribution for that variable. Operating in log space we add this arrays of probabilities to come up with the total distribution.

```python Log Distribution for a Variable Given an Assignment
def logDistAss(self, sample_var, evidence):
    cardio = set([e.totCard() for e in sample_vars])
    logbp = [0.]*cardio.pop()
    for fu in self.var_to_factors[sample_var]:
        evi = dict((k, evidence[k]) for k in fu.variables if k!= sample_var)
        fu = FactorOperations.reduce_to_dist(fu, evi)
        logbp += np.log(fu)
    logbp = logbp - min(logbp) # to avoid underflow when back in normal space
    return logbp
```

Note three things:

* that FactorOperations.extract is an added method that works the same as marginalize but marginalizing in a sweep all variables except the one to extract.
* .var_to_factors is a dictionary where for each variable we have a set of the factors it is involved with. This structure is computed once and allows us to compute faster the sampling process.
* to speed up things I have coded a utility function based on reducing a factor by evidence. It gets an assignment to all factor variables except one and produces an array with the distribution resulting for that variable.

```python Reducing Factor to a single var distribution
def reduce_to_dist(fA, evidence):
    '''reduce factor by complete assignment to a single variable distribution'''
    ass = []
    counter = 0
    for x in fA.variables:
        if x in evidence:
            ass.append(evidence[x])
        else:
            counter += 1
            ass.append(slice(None))
    if counter != 1:
        raise Exception("Error in observed vars")
    return fA.values[ass]
```

## One step down the Chain

The Gibbs sampling process, starts with a complete assignment and ends with another complete assignment. For each variable we:

* compute the distribution based on the other assigned variables, 
* find it's new assignment based on rolling a dice on the distribution,
* update the assignment with this new value so for the new iteration one variable of the assignment has changed

 We repeat for all variables, updating all variables in the process, until we get a newly complete assignment that corresponds to the new state in the Markov Chain.

```python Gibbs 1 step down the Markov Chain
def Gibbs(self, ass):
    for x in self.G.v:
        p = self.logDistAss([x], ass)
        ass[x] = self.randSampleDist(x.totCard(), exp(p))
    return ass
```

## Running around the Chain

Collecting samples from the Markov Chain is as easy as running iteratively the previous Gibbs method. We introduce two new inputs:

* mix_time: how many steps to take before we think the chain has converged and we start collecting samples. Consider that at the start we are wondering near the initial assignment and to reach a 'real' assignment we need enough time walking around.
* num_samples: after mixing time, how many steps to take, how many samples to collect.

The code is self explanatory:


```python Collecting Samples from the Makov Chain
def sampleMarkovChain(self, mix_time, num_samples, seed_assignment):
    max_iter = mix_time + num_samples
    all_samples = np.array([seed_assignment[e] for e in sorted(seed_assignment)])
        
    for i in range(max_iter):
        seed_assignment = self.Gibbs(seed_assignment)
        sol = [seed_assignment[e] for e in sorted(seed_assignment)]
        if i > mix_time:
            all_samples = np.vstack((all_samples, sol))
    return all_samples
```

## Inference

Now that we have the samples we can compute the estimated marginals by counting, for each variable, how many times each as assignment shows up.

```python Inference from MCMC using Gibbs Transitions
def infere(self, mix_time, num_samples, seed_assignment):
    all_samples = self.sampleMarkovChain(mix_time, num_samples, seed_assignment)

    M = [None]*self.G.V
    for i,x in enumerate(self.G.v):
        M[i] = Factor.Factor([x])
        M[i].fill_values(FactorOperations.normalize(np.bincount(all_samples[:,i]).astype(np.float)))    
    return M
```
