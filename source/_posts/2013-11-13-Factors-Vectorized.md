---
layout: post
title: "Factors Vectorized"
date: 2013-11-13 08:01
comments: true
categories: PGN
---

I have vectorized the Factor implementation, getting rid of loops and using as many ndarray operations as possible. To begin with we have a basic Factor class that still holds variables, cardinalities and values. The main difference is that instead of a 1D array for the values we have an ndarray with as many dimensions as variables.

```python Factor objects
class Factor(object):
    '''Factor : CPD Conditional Probability (discrete) Distribution
      var    Vector of variables in the factor, e.g. [1 2 3] => X1, X2, X3. always ordered by id
      card   Vector of cardinalities of var, e.g. [2 2 2] => all binomials
      val    Value table with the conditional probabilities. n dimensional array
      size   Length of val array = prod(card)'''
    
    def __init__(self, variables):
        self.variables = variables
        self.cards = [c.totCard() for c in self.variables]
        self.values = np.zeros(self.cards).astype('float64')
    
    def idx2ass(self, arr):
        '''mapping between all combinatorial ordered assignments and a flatten vector of values'''
        t = [[e]*len(self.cards) for e in range(len(arr))]
        order, dic = [], {}
        for q, row in enumerate(t):
            prod, ass = 1, []
            for i, e in enumerate([1] + self.cards[:-1]):
                prod *= e
                ass.append((row[i]/prod) % self.cards[i])
            key = tuple(ass)
            dic[key] = arr[q]
            order += [key]
        return [order, dic]

    def fill_values(self, arr):
        order, dic = self.idx2ass(arr)
        for e in order:
            self.values[e] = dic[e]
```

The idx2ass is a holdout from the previous version that helps us understand the assignment process, allows us to fill the values with a 1D array and, is useful when comparing with the Octave implementation for debugging purposes. This method returns two sets that can be used in many ways as idx2ass and ass2idx. Much of this code can be simplified and cleaned further.

## Marginalization

The key advantage of using an ndarray with variables mapped to axes (dimensions) is that marginalize becomes a trivial operation. We only need to sum up all the values of the marginalized variable.

```python Factor Reduction
def marginalize(fA, v):
    new_vs = [e for e in fA.variables if e != v ]
    if new_vs==[]:
        raise Exception("Error: Resultant factor has empty scope")
    f = Factor.Factor(new_vs)
    pos_m = fA.variables.index(v)
    f.values = np.sum(fA.values, axis=pos_m)
    return f
```
For example, for a factor with two random variables (v1, v2), reducing on v2 means selecting the axis for v2 and for each row of v1, adding up all columns of v2.

{% img center /images/nov13/margin.png %}

## Conditioning

Also trivial, we modify the ndarray values by 

- focusing on the observed variable axis and leaving all other axis untouched
- for the selected axis, setting to 0. all cells that are not in the observation column

```python Conditioning on evidence
def observe(fA, evidence, norma=True):
    f = Factor.Factor(fA.variables)
    M = fA.values.copy()
    for cond_var in evidence.keys():
        if cond_var in fA.variables:
            a = []
            for v in fA.variables:
                if v.id == cond_var.id:
                    a += [[x for x in range(v.totCard()) if x!=evidence[cond_var]]]
                else:
                    a += [slice(None)]
            M[a]=0.
    f.values = normalize(M) if norma else M
    return f
```

Conditioning renormalizes at the end to ensure proper probabilities. 

## Multiplication

The difficult one. Given two factors I modify each one by:

- adding all the variables of the resulting multiplication factor (Union of all variables, sorted by id)
- each added variable inserts a new axis in the right order
- we expand the values ndarray on each new axis by repeating a number of times equal to the cardinality of the axi's variable.

Continuing with the graphic example, to expand our previous factor (variables v1 and v2) by another variable (v3) we would start with the 2D values along axes v1, v2. Then we add a third dimension for v3.

{% img center /images/nov13/multiply1.png %}

And then we repeat the 2D matrix (v1,v2) along the v3 axis. In our case v1 and v2 have cardinality 2 and v3 has cardinality 3 so we repeat the 2D matrix twice more along the v3 axis.

{% img center /images/nov13/multiply2.png %}

At the end of the process we have two ndarrays that represent the same variables, aligned and of the same shape. To multiply we only need to multiply elementwise. Convince yourself with the following example:

```text basic example
factor_1(v1) with values [1,2] => expand to [[1,1][2,2]] inserting v2 and resulting in (v1, v2)
factor_2(v2) with values [3,4] => expand to [[3,4][3,4]] inserting v1 and resulting in (v1, v2)

factor_1_expanded x factor_2_expanded = factor_product
[1, 1]            x [3, 4]            = [3, 4]
[2, 2]              [3, 4]              [6, 8]
```


We end by renormalizing to ensure proper probabilities.

```python Factor product
def realign(M, chaos):
    '''realign by swapping its axes'''
    order = range(len(chaos))
    for i in order:
        have_here = chaos[i]
        should_be = order[i]
        index_of_missing_should = chaos.index(should_be)
        if have_here != should_be:
            M = np.swapaxes(M, i, index_of_missing_should)
            chaos[index_of_missing_should] =  have_here
            chaos[i] = should_be
    return M

def new_axes(whole, partial):
    '''list of new axis needed for partial to become whole'''
    sol = []
    j = 0
    for e in whole:
        if e in partial:
            sol += [j]
            j += 1
        else:
            sol += [np.newaxis]
    return sol

def expand_rvars(whole, fA):
    '''expand a factor to new dimensions by inserting new axes and repeating values along them'''
    f = Factor.Factor(whole)

    chorizo = fA.values.copy()
    order = sorted(fA.variables)
    qt = [order.index(e) for e in fA.variables] # [0,1,3,2] indexes of fA.vars vs order [0,1,2,3]
    if qt!=range(len(qt)):                      # if not equal, we need to realign before proceeding
        chorizo = realign(chorizo, qt)

    sol = new_axes(whole, order)                # see which new axis we need
    for i,e in enumerate(sol):
        if e==None:                             # insert new axes and fill values
            chorizo = np.expand_dims(chorizo, axis=i)
            chorizo = np.repeat(chorizo, whole[i].totCard(), axis=i)    
    f.values = chorizo
    return f

def multiply(fA, fB, norma=True):
    '''expanding factors to the whole set of variables, then we can multiply element-wise'''
    allVars = sorted(list(set(fA.variables) | set(fB.variables)))
    f = Factor.Factor(allVars)
    FA = expand_rvars(allVars, fA)
    FB = expand_rvars(allVars, fB)
    f.values = FA.values * FB.values
    if norma:
        f.values = normalize(f.values)
    return f
```

As always this code is a first pass that needs cleaning and refactoring (most probably I can still simplify the subprocesses for the expansion of factors).









