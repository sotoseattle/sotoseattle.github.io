---
layout: post
title: "PRIMO Genes with Clique Trees"
date: 2014-07-20 8:01
comments: true
categories: PGN, Primo
published: false
---

Now we revisit our revised Cystic Fibrosis Simple Genetic Network from two days ago [post](/blog/2014/07/19/Grim-Genes/) and from our previous python code [post](/blog/2014/01/21/Gene-Clique-1/). Instead of inferring marginals by computing the overall factor product over all the variables (which we saw is too exhaustive and intensive), we will build the associated clique tree and use Belief Propagation.

All the code is included at [[Primo @ Github](https://github.com/sotoseattle/Primo)]. Our initial network was:

{% img center /images/nov13/cysticBN.png 600 Template for Genetic Bayesian Network%}

The clique tree looks like this:

{% img center /images/jan14/cystic_BN_clique.png 600 Clique Tree for Genetic Bayesian Network%}

We need some back-office stuff to organize the genetic data:

```ruby Framework
class Phenotype < RandomVar
  def initialize(id, name)
    super(id, 2, "#{name}_ph", ['present', 'absent'])
  end
end

class Genotype < RandomVar
  def initialize(id, name)
    super(id, 3, "#{name}_gn", ['FF', 'Ff', 'ff'])
  end
end

class Person
  attr_reader :id, :name, :genotype, :phenotype, :f_phenotype, :f_genotype, :parent_1, :parent_2

  def initialize(id, name)
    @name = name
    @phenotype = Phenotype.new(id, name)
    @genotype = Genotype.new(id+1, name)
    @f_phenotype = Factor.new([@phenotype, @genotype], [0.8, 0.2, 0.6, 0.4, 0.1, 0.9])
    @f_genotype = Factor.new([@genotype], [0.01, 0.18, 0.81])
  end

  def is_son_of(parent_1, parent_2)
    @parent_1, @parent_2 = parent_1, parent_2
    na = ...
    @f_genotype = Factor.new([@genotype, @parent_1.genotype, @parent_2.genotype], na)
  end

  def observe(ass, var_type)
    rv = instance_variable_get("@#{var_type}")
    f = instance_variable_get("@f_#{var_type}")
    f = f.reduce!({rv=> ass}, true)
  end
end

class Family
  attr_reader :members, :clique_tree

  def initialize(names)
    @clique_tree = nil
    @members = []
    i = 0
    names.each do |name|
      @members << Person.new(i, name)
      i +=2
    end
  end

  def [](name)
    @members.find{|p| p.name==name}
  end

  def setup
    all_factors = @members.map{|p| [p.f_phenotype, p.f_genotype]}.flatten
    @clique_tree = CliqueTree.new(all_factors)
    @clique_tree.calibrate()
  end

  def query(name, var_type)
    rv = self[name].instance_variable_get("@#{var_type}")
    b = @clique_tree.betas.find{|b| b.vars.include?(rv)}
    b.marginalize_all_but!(rv)
    b = b.normalize!.vals.to_a
    sol = {}
    rv.card.times do |i|
      sol[rv.ass[i]]=b[i]
    end
    return sol
  end
end
```

As before, we observe some variables and the code correctly computes the exact marginals for all phenotype variables in  0.3s, half the time the python code needed. The simplicity of the computation (just two passes up and down the tree) allows us to make the net as big as desired.

```python Running Inference Engine
a = Family.new(%w{Ira Robin Aaron Rene James Eva Sandra Jason Benito})

a['James'].is_son_of(a['Ira'], a['Robin'])
a['Eva'].is_son_of(a['Ira'], a['Robin'])
a['Sandra'].is_son_of(a['Aaron'], a['Eva'])
a['Jason'].is_son_of(a['James'], a['Rene'])
a['Benito'].is_son_of(a['James'], a['Rene'])

a['Ira'].observe('present', 'phenotype')
a['Rene'].observe('FF', 'genotype')
a['James'].observe('Ff', 'genotype')

a.setup()

%w{Ira Robin Aaron Rene James Eva Sandra Jason Benito}.each do |name|
  q = a.query(name, 'phenotype')
  puts "#{name} p(showing illness) = #{100*q['present']}%"
end

```
