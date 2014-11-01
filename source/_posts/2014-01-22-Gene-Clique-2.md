---
layout: post
title: "Gene Net Decoupled & BP Inference"
date: 2014-01-22 08:01
comments: true
categories: PGN, theory
---

And now we revisit our revised [Decoupled Genetic Network](/blog/2013/11/06/GBN4) for cystic fibrosis and instead of infering marginals by computing the overal factor product over all the variables (which we saw even more exhaustive and intensive than the basic network), we will build the associated clique tree and use Belief Propagation.

Our initial network was:

{% img center /images/nov13/decoup_cysticBN.png 600 Template for Decoupled Genetic Bayesian Network%}

The clique tree looks like this:
{% img center /images/jan14/cystic_decoup_BN_clique.png 600 Clique Tree for Decoupled Genetic Bayesian Network%}

As before, we observe some variables and the code correctly computes the exact marginals for all phenotype variables in under a second. Consider that before the computer took minutes to compute it.

```python Running Inference Engine
[F, v] = geneticNetwork(family_tree, frequency_of_alleles_in_general_population, probability_of_trait_based_on_genotype)

cc = CliqueTree.CliqueTree(F)

# for fun lets reduce some evidence
cc.factors[2] = FactorOperations.observe(cc.factors[2], {v[17]:0}) # Ira shows pheno
cc.factors[5] = FactorOperations.observe(cc.factors[5], {v[6]:0})  # rene has gen1 F
cc.factors[5] = FactorOperations.observe(cc.factors[5], {v[7]:1})  # rene has gen2 f
cc.factors[1] = FactorOperations.observe(cc.factors[1], {v[5]:0}) # Eva shows pheno

cc.calibrate()

# compute exact marginals for phenotypes
phenos_nodes = [0,1,2,3,4,5,6]
probs = {}
for i in phenos_nodes:
    belief = cc.beta[i]
    genes = [v1 for v1 in belief.variables if not v1.id.endswith("_p")]
    f = copy.copy(belief)
    for g in genes:
        f = FactorOperations.marginalize(f, g)
    f.values = FactorOperations.normalize(f.values)
    probs[f.variables[0].id] = f.values[0]
print probs
=> {'Jason_p': 0.54095556127444155, 'James_p': 0.47255963793489481, 'Ira_p': 1.0, 'Rene_p': 0.59999999999999998, 'Benito_p': 0.54095556127444155, 'Robin_p': 0.42462267673013965, 'Eva_p': 1.0}
```
