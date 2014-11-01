---
layout: post
title: "Gene Network & BP Inference"
date: 2014-01-21 08:01
comments: true
categories: PGN, theory
---

We revisit our initial [Genetic Network](/blog/2013/11/04/GBN2) for cystic fibrosis and instead of infering marginals by computing the overal factor product over all the variables (which we saw was exhaustive and intensive), we will build the associated clique tree and use Belief Propagation.

Our initial network was:

{% img center /images/nov13/cysticBN.png 600 Template for Genetic Bayesian Network%}

And its built factors were:

```python All Factors of Genetic Net
v = {}
v[0] = Rvar.Rvar('rene_g', 3)
v[1] = Rvar.Rvar('rene_p', 2)
...
v[12] = Rvar.Rvar('james_g', 3)
v[13] = Rvar.Rvar('james_p', 2)
...
v[16] = Rvar.Rvar('robin_g', 3)
v[17] = Rvar.Rvar('robin_p', 2)

f = [None]*18
# Robin
f[0] = Factor.Factor([v[17], v[16]])
f[0].fill_values([0.8, 0.2, 0.6, 0.4, 0.1, 0.9])
f[1] = Factor.Factor([v[16]])
f[1].fill_values([0.01, 0.18, 0.81])

# James
f[4] = Factor.Factor([v[13], v[12]])
f[4].fill_values([0.8, 0.2, 0.6, 0.4, 0.1, 0.9])
f[5] = Factor.Factor([v[12], v[14], v[16]])
f[5].fill_values([1., 0., 0., 0.5, 0.5, 0.0, 0.0, 1.0, 0., 0.5, 0.5, 0., 0.25, 0.5, 0.25, 0., 0.5, 0.5, 0., 1., 0., 0., 0.5, 0.5, 0., 0., 1.])
...
```

The clique tree for belief propagation is built in a single line:

```python Voilá
cc = CliqueTree.CliqueTree(f)
```
And looks like this:
{% img center /images/jan14/cystic_BN_clique.png 600 Clique Tree for Genetic Bayesian Network%}

As an example, the following code states the observations and correctly computes the exact marginals for all phenotype variables in under a second.

```python Reduce and Compute
cc.factors[3] = FactorOperations.observe(cc.factors[3], {v[15]:0}) # Ira shows pheno
cc.factors[6] = FactorOperations.observe(cc.factors[6], {v[0]:0})  # Rene has gen FF
cc.factors[4] = FactorOperations.observe(cc.factors[4], {v[12]:1}) # James has gen Ff

cc.calibrate()

phenos_nodes = [0,1,2,3,4,5,6,7,8]
probs = {}
for i in phenos_nodes:
	belief = cc.beta[i]
	genes = [v for v in belief.variables if not v.id.endswith("_p")]
	f = copy.copy(belief)
	f = FactorOperations.marginalize(f, genes[0])
	f.values = FactorOperations.normalize(f.values)
	probs[f.variables[0].id] = f.values[0]
print probs
=> {'jason_p': 0.69999999999999996, 'ira_p': 1.0, 'james_p': 0.59999999999999998, 'rene_p': 0.80000000000000004, 'benito_p': 0.69999999999999996, 'robin_p': 0.24155844155844158, 'eva_p': 0.40720779220779224, 'aaron_p': 0.19700000000000001, 'sandra_p': 0.30061363636363636}
```