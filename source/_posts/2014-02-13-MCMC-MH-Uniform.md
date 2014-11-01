---
layout: post
title: "MCMC MH Uniform"
date: 2014-02-13 8:01
comments: true
categories: PGN, 
---

The code for a Q based on the Uniform distribution has three steps:

* come up with a new complete assignment out of the blue (literally)
* compute the acceptance probability. When we compute π(x')Q(x' → x), π is the joint probability reduced by the complete assignment (unormalized), meaning the total probability of having that assignment. The question is, how do I compute Q(x' → x)? Since it is the uniform distribution, Q(x' → x) == Q(x → x') == constant, so it cancels out and I only need to compute the ratio of π(x') / π(x).
* decide with the new acceptance if, like The Clash asks, should we stay or should we go.

```python Q from Uniform dist
def MH_Uniform(self, from_ass):
    # get a random complete assignment from uniform distribution
    cardio = [x.totCard() for x in self.G.v]
    a = floor([uniform(0., 1.)*cardio[i] for i in range(len(cardio))]).astype(np.int)
    to_ass = dict((e,a[i]) for i,e in enumerate(self.G.v))
        
    # compute the probabilities of transitions
    fwd, bwd = 0.0, 0.0
    for fu in self.G.factors:
        fwd += math.log(FactorOperations.reduce_to_value(fu, to_ass))
        bwd += math.log(FactorOperations.reduce_to_value(fu, from_ass))
        
    # compute acceptance probability and return the new state
    p_acceptance = min([1., (math.exp(fwd)/math.exp(bwd))])

    if random.random() < p_acceptance:
        from_ass = to_ass
    return from_ass
end
```

To speed up the process, I have added a new utility function to Factor Operations, to extract the potential/probability value of a complete assigment from a given factor. It is the same as reducing, but instead of giving back a reduced factor, since we have a complete assignment (all variables have assignments) we return a value (float).

```python Get Factor Value of Complete Assignment
def reduce_to_value(fA, evidence):
    '''reduce factor by complete assignment to a single value'''
    complete_ass = tuple([evidence[x] for x in fA.variables])
    return fA.values.item(complete_ass)
```

## Mixing

As before we run the MCMCMHU to collect a total of 70,000 samples. The initial assignment is initialized to all ones. 

Now we get subsets every 1,000 samples of size 10,000. That is 59 windows of 10,000 samples collected after different mixing times.

Ploting the difference between the exact marginal and our estimated marginals of the three variables (number 0 as red, 4 as blue, and 8 as green), we have the following charts (from two different runs).

<div>
	{% img left /images/feb14/MHU_mix.png 425 Mixing Windows Run 1%}
	{% img right /images/feb14/MHU_mixB.png 425 Mixing Windows Run 2%}
</div>
<br/>

No clear mixing threshold.

## Sample size

Following the previous settings but fixing the mixing time to 1,000, we can plot the difference between exact and estimated marginals as we collect more samples. From the charts (two different runs) we see that it converges beyond 20,000. 

<div>
	{% img left /images/feb14/MHU_size.png 425 Sample Size Run 1%}
	{% img right /images/feb14/MHU_sizeB.png 425 Sample Size Run 2%}
</div>	
<br/>

Nevertheless, it seems to work worse than Gibbs.

## Marginals Convergence

We again run twice the model and see how well the estimated marginals converge across runs. All models are Ising grids with 9 variables, have mixing time of 7,000 and sample size of 10,000. 

<div>
	{% img left /images/feb14/MHU_compare_runs_0703.png 425 %}
	{% img right /images/feb14/MHU_compare_runs_102.png 425 %}
	{% img center /images/feb14/MHU_compare_runs_095005.png 425 %}
</div>

Again, worse results.


## Comparison to Gibbs

Lile we did for Gibbs, we run 10 times this MCMCMHU model. Now with mixing time of 5,000 and sample size of 30,000. Starting assignment, 5 runs with 0s, and 5 runs with 1s. We compute the error as before and see what results we get as compared to the exact marginals for all variables.

The error, as defined in previous posts, from our 10 runs is 0.12 on the average (vs. 0.065 from Gibbs), with a range of [0.03, 0.24] (vs. [0.005, 0.17]) and a standard deviation of 0.07 (vs. 0.05). Definetively worse performance.

Finally, this is for a 9 node Ising net. For a 4 sided net with 16 nodes, with strong correlation this MHU method performs very poorly because we get stuck in specific assignments for very long times. Because the probability of a new assignment (randomly chosen) tends to be so small as compared to the probability we start with that the ratio is miniscule, and A() always rejects. The more correlated (i.e. higher on diagonal values vs off diagonal) the more steps we need to take, and even that doesn't guarantee good results. Consider that for 100,000 steps, only around 50 are accepted, which means we need to run it much longer to achieve good results. 

For the easiest 16 nodes with pairwise potentials [0.5, 0.5], 0 mixing and 30,000 steps, MHU and Gibs give similar good results. MHU gives us an average error and standard dev of [0.02, 0.004], while Gibbs achieves [0.01, 0.001].

<div>
    {% img left /images/feb14/Gibbs_hist.png 425 Gibbs Error 100 Runs%}
    {% img right /images/feb14/MHU_hist.png 425 MHU Error 100 Runs%}
</div>
<br/>

Finally we compare the histogram of errors for these two methods when applied to the [1.0, 0.2] in the 16 node Ising model. Each histogram is based on 100 runs. Both have a mixing time of 10,000 and a sampling size of 70,000. Gibbs's histogram shown in red, MHU in blue.
