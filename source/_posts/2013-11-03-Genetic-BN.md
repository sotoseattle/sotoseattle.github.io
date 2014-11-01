---
layout: post
title: "Genetic Bayesian Network (I)"
date: 2013-11-03 08:01
comments: true
categories: PGN, genetic
---

[UPDATE] I have updated a posteriori the code to include the vectorized version of Factor manipulations, which is incredibly faster and will be explained in a later post.

As a first exercise I am going to create the typical genetic Bayesian network. Given a family tree, for each member of the family we are going to have two nodes, one for the person's genotype, and another for her phenotype. The template would be:

{% img center /images/nov13/template.png 500 Template for Genetic Bayesian Network%}


## phenotype Factor
The Phenotype of a person will depend on her Genotype, so there is a single link between both nodes. The factor that describes this relationship (probability of trait present given the genotype) will be a function that computes the probability of each possible assignment. 

The phenotype can be either present (val 0) or not (val 1) with a cardinality of 2.

In the simplest case the genotype has 2 alleles and each one can be dominant (A) or recessive (a). The presence of the dominant allele (A) determines the presence of the trait. 

This means a total of 3 possible allele combinations: AA, Aa\=\=aA, aa. For example, given a genotype x\_3, the conditional probabilities of phenotype x\_1 showing up are:


|-------------|----|---|---|---------------------------------|
| |||||
|:------------|---:|--:|--:|:--------------------------------|
|-------------|----|---|---|---------------------------------|
|p(x\_1 \| x\_3)| AA | Aa| aa||
|present      |  1 |  1|  0|                                 |
absent        |  0 |  0|  1||
|-------------|----|---|---|---------------------------------|
{:.widetable}


<br/>
In a fast and dirty way the above becomes:

```python Factor for phenotype given genotype in Mendellian manner
geno  = Rvar.Rvar(1, 3)
pheno = Rvar.Rvar(2, 2)

def Factor_phenotype_given_genotype_Mendelian(isDominant, genoVar, phenoVar):
    varis = [phenoVar, genoVar] # written like in cond prob order
    if isDominant:
        valus = [1., 0., 1., 0., 0., 1.]
    else:
        valus = [0., 1., 1., 0., 1., 0.]
    f = Factor.Factor(varis)
    f.fill_values(np.array(valus))
    return f
```

Generalizing the previous case we can determine probabilities for the presence of the trait. For this we supply an alphaList that tells us the probability of presenting the trait if certain genotype is present. For example, continuing the previous example, and alphaList of [0.8, 0.6, 0.1] that correlates with genotypes [AA, Aa, aa], means that if the AA is present, the probability of having the trait is 80%.

```python Factor for phenotype given genotype and probabilities
def Factor_phenotype_given_genotype_Not_Mendelian(alphaList, genoVar, phenoVar):
    ff = Factor.Factor([phenoVar, genoVar]) # written like in cond prob order
    order, dic = ff.idx2ass(ff.values.flatten())
    arr = ff.values.copy().flatten()
    for i in range(len(alphaList)):
        v = alphaList[i]
        k = order.index((0,i))  # trait present
        arr[k] = v
        arr[k+1] = 1.-v    # trait absent
    ff.fill_values(arr)
    return ff
```

## genotype Factor without inheritance

The simplest Factor for genotype is the one that does not depend on inheritance and instead depends solely on the frequency of the alleles showing up in the general population. 

We consider these probabilities to be independent. As a result, the probability of a genotype is the product of the frequencies of its constituent alleles.

```python Factor Genotype given Alleles frequency (in general population)
def genes2Alleles(numAlleles):
    sol = {}
    count = 0
    for i in range(numAlleles):
        for j in range(i, numAlleles):
            sol[(i,j)] = count
            count +=1
    return sol

def Factor_genotype_given_allele_freqs(alleleFreqs, genoVar):
    ff = Factor.Factor([genoVar])
    
    numAlleles = len(alleleFreqs)    
    numGenotypes = genoVar.totCard()
    chocho = genes2Alleles(numAlleles)
    for row in range(numAlleles):
        for col in range(numAlleles):
            key = chocho[tuple(sorted((row, col)))]
            ff.values[key] += alleleFreqs[row]*alleleFreqs[col]
    return ff
```

## genotype Factor without inheritance

Finally, the probabilities of having a certain genotype based on the genotypes of the parents:



```python Factor Genotype given Parents Genotypes
def numGenos(numAlleles):
    return numAlleles*(numAlleles+1)/2

def Factor_genotype_given_parents_genes(numAlleles, genKid, genDad, genMom):
    ff = Factor.Factor([genKid, genDad, genMom])
    order, dic = ff.idx2ass(ff.values.flatten())
    chocho = genes2Alleles(numAlleles)
    kid = chocho
    dad = chocho.copy()
    mom = chocho.copy()
    arr = ff.values.flatten()
    for dad_gen in dad:
        for mom_gen in mom:
            for kid_gen in kid:
                hits = 0.
                for chuchu in itertools.product(dad_gen, mom_gen):
                    if sorted(chuchu)==list(kid_gen):
                        hits += 1
                hits = hits/4.
                arr[order.index((kid[kid_gen], dad[dad_gen], mom[mom_gen]))] = hits
    ff.fill_values(arr)
    return ff
```

## building the genetic network

Self explanatory. For a given family tree we create the variables and factors associated to each person/node.


```python putting it all together
def geneticNetwork(pedigree, alleleFreqs, alphaList):
    PHENOTYPE_CARD = 2 # present or absent
    numAlleles = len(alleleFreqs)
    numG = numGenos(numAlleles)
    
    varList = {}
    
    # first we create all variables
    count = 0
    for name in pedigree:
        varList[name] = {}
        varList[name]['var_geno'] = Rvar.Rvar(count, numG)
        count +=1
        varList[name]['var_pheno'] = Rvar.Rvar(count, PHENOTYPE_CARD)
        count +=1
        
    # and now the factors
    for name in pedigree:
        kid_gn = varList[name]['var_geno']
        kid_ph = varList[name]['var_pheno']
        
        dad_gn, mom_gn = pedigree[name]
        if dad_gn and mom_gn:
            dad_gn = varList[dad_gn]['var_geno']
            mom_gn = varList[mom_gn]['var_geno']
            ff = Factor_genotype_given_parents_genes(numAlleles, kid_gn, dad_gn, mom_gn)
        else:
            ff = Factor_genotype_given_allele_freqs(alleleFreqs, kid_gn)
        varList[name]['factor_geno'] = ff
        varList[name]['factor_pheno'] = Factor_phenotype_given_genotype_Not_Mendelian(alphaList, kid_gn, kid_ph)
    
    return varList

# this is fugly
def modify_Factor_by_evidence(name, node, ass):
    factor = GN[name]['factor_'+node]
    randvar = GN[name]['var_'+node]
    GN[name]['factor_'+node] = FactorOperations.observe(factor, {randvar:ass})

def build_joint_cpd():
    a = None
    for k in GN.keys():
        b = FactorOperations.multiply(GN[k]['factor_geno'], GN[k]['factor_pheno'])
        if a==None:
            a = b
        else:
            a = FactorOperations.multiply(a, b)
    return a
```


















