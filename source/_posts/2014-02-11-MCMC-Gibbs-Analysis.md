---
layout: post
title: "MCMC Gibbs - Analysis"
date: 2014-02-11 8:01
comments: true
categories: PGN, theory
---

We are going to use our primitive MCMC Gibbs Inference engine on our 9 variable toy Ising grid (with histerical potentials [1.0, 0.2]). And then compare our results to our previous approximation using Loopy Belief Propagation.

{% img center /images/feb14/toy_grid.png 300 Small Pairwise Markov Grid%}

The big problem is that there is no clear cut way to know when we have left the wandering zone around and entered the domain of the joint probability. We cannot know when the chain has mixed, we can only guess.

## Mixing

As a first experiment we run our MCMC to collect a total of 70,000 samples. The initial assignment is initialized to all ones. 

Now from the samples we get subsets every 1,000 samples of size 10,000. That is 59 subsets that go from sample [0 to 10,000], [1,000 to 11,000], [2,000 to 12,000], .... Each subset is a window, of 10,000 samples collected after different mixing times.

If we plot the difference between the exact marginal and our MCMC estimated marginal of the three variables (number 0 as red, 4 as blue, and 8 as green), we have the following charts (from two different runs).

<div>
	{% img /images/feb14/dist_to_exact_mcmcg.png 425 Run 1%}
	{% img right /images/feb14/dist_to_exact_mcmcgB.png 425 Run 2%}
</div>


What I deduce from this (and here is pure amateur talking) is that all variables tend to move together. I don't know what is a good mixing time, it looks like it is pretty much mixed from the start. Nevertheless, I'll choose 7,000 to be safe. Also, as we walk around, the system oscilates and spike away from the exact marginals in a bounded range.


## Sample size

Following the previous settings but fixing the mixing time to 7,000, we can plot the difference between exact and estimated marginals as we collect more samples. From the charts (two different runs) we see that anything beyond 30,000 samples seems to be ok.

<div>
	{% img left /images/feb14/dist_to_exact_mcmcg_3.png 425 Run 1%}
	{% img right /images/feb14/dist_to_exact_mcmcg_3B.png 425 Run 2%}
</div>	

## Marginals Convergence

Gibbs walks one step at a time, so for grids with high potentials (high correlation between variables), it tends to stay for long periods next to high potential states (or clusters), slowing down mixing.

In the following charts we run twice different models and see how well the estimated marginals converge across runs. All models are Ising grids with 9 variables, have mixing time of 7,000 and sample size of 10,000. 

<div>
	{% img left /images/feb14/compare_runs_0703.png 425 %}
	{% img right /images/feb14/compare_runs_102.png 425 %}
	{% img center /images/feb14/compare_runs_095005.png 425 %}
</div>	

The better convergence the closer the estimated marginals to the diagonal. The first model has potentials [0.7, 0.3] and all variables are along the diagonal. The same happens for the second model, our optimized [1., 0.2], where the a corner variable get a bit off, but good for all purposes. 

A third model with even further heighened potentials [0.95, 0.05] shows discouraging results. The true marginals are all in the narrow range [0.5974, 0.6009], but our estimated marginals for the first run are tightly clustered around 0.0, while all marginals for the second run are tightly clustered around 0.6. This can be palliated with a bigger sample size up to a point.

Two points to make: 1) this shows convergence between runs, not to the exact marginal. 2) The lower correlation between variables, the better convergence.

## Comparison to LBP

Finally we are going to run 10 times the model we have. Mixing time of 7,000. Sample size of 20,000. Starting assignment, 5 runs with 0s, and 5 runs with 1s. We are going to compute the error as before and see what results we get as compared to the exact marginals for all variables.

The error, as defined in previous posts, from our 10 runs is 0.065 on the average, with a range of [0.005, 0.17] and a standard deviation of 0.05. That is a further improvement from our best approximation of 0.24 with RBP.

Nevertheless, although Gibbs gives us on the average much better results, from time to time it go cyclothymic and spikes markedly away from the exact marginals.

As a data point for comparison purposes, taking the estimated marginals from our last run (the 10th run) we get an error of 0.059, and the already familiar table:

|var| exact marginal | MCMC Gibbs aprox|
|:-:|---------------:|---------------:|
||||
|0 | 0.537310955208 | 0.52015|
|1 | 0.556558936419 | 0.541|
|2 | 0.552439017812 | 0.53745|
|3 | 0.573509953445 | 0.5518|
|4 | 0.6            | 0.57795|
|5 | 0.607090867757 | 0.58745|
|6 | 0.611810074317 | 0.5918|
|7 | 0.621715017707 | 0.59885|
|8 | 0.626241528067 | 0.60425|
{:.widetable}
<br/>

Even for higher order Ising grids, as long as we don't have way too highly correlated variables, we mix long enhough and we collect a sufficiently big sample size, we get pretty decent results!