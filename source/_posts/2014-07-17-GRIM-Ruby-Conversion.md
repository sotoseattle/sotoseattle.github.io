---
layout: post
title: "GRIM: Ruby Conversion"
date: 2014-07-17 8:01
comments: true
categories: PGN
published: false
---

I have decided to code all this in Ruby instead of Python. The reason being that Ruby is more flexible in order to build prototypes that encompass all aspects of an app. The apps that I want to build around ML are like cars. The engine may be a key component and its performance paramount but there is more to a car than its engine. In the same manner, our apps will be more than its inference engine, and in all those aspects Ruby shines. If I was obsessed only with the engine I would code it all in Julia or C, but I am in this for the fun and the possibilities, and Ruby is a pleasure to play with.

Besides, what I have already coded in Ruby is already faster than my Python code (which shows what a beginner I am in Python). The key to the performance boost has been the use of the [NArray gem from Masahiro Tanaka](http://masa16.github.io/narray/), which allows me to, for example, multiply two multidimensional arrays element wise in a single step, after aligning them with simple rotations of their axes (actually pretty cool).

I have christened this working library as GRIM (Graphical Inference Modeling with Ruby), and the code is available at https://github.com/sotoseattle/GRIM

It works along the same lines as the Python version so the following posts will be a bit repetitive since we are rehearsing much of what was done months ago in Python. This is an ongoing effort that will finish in a few weeks and that I hope to culminate with a working CRF application that estimates its own parameters.

The Factor class now holds all factor operations as bang! instance methods.

```ruby Factor class
class Factor

  attr_reader :vars, :vals, :cards

  def initialize(variables, arr=nil)
    @vars = Array(variables)
    @cards = @vars.map{|e| e.card}
    arr ? load_vals(arr) : reset_vals

    raise ArgumentError.new() if @vars.empty?
    raise ArgumentError.new() unless @vars.all?{|e| e.class<=RandomVar}
  end

  def [](assignments)
    indices = []
    Array(assignments).each_with_index{|s,i| indices << @vars[i].ass.index(s)}
    return @vals[*indices]
  end

  def load_vals(arr)  # fill values matrix from array data
    @vals = NArray.to_na(arr).reshape!(*@cards)
  end

  def reset_vals
    @vals = NArray.float(*@cards)
  end
  ...

end
```

## Marginalization

As in Python we just add up along the chosen axis.

```ruby Marginalization
def marginalize!(variable)
  raise ArgumentError.new() unless variable.class<=RandomVar
  new_vars = @vars.reject{|e| e==variable}
  unless new_vars.empty?
    axis_to_marginalize = @vars.index(variable)
    @vars = new_vars
    @cards = @vars.map{|e| e.card}
    @vals = @vals.sum(axis_to_marginalize)
  end
  self
end
```

For example, for a factor with two random variables (v1, v2), reducing on v2 means selecting the axis for v2 and for each row of v1, adding up all columns of v2.

{% img center /images/nov13/margin.png %}

## Reduction

Conditioning loops a bit, but this operation is not critical nor intensively used. It is usually called at input time to reduce the factors that we work with, so it is not a big deal if it is not vectorized.

We modify the NArray values by 1) selecting the observed variable axis and leaving all other axis untouched, and 2) for the selected axis, setting to 0. all cells that are not in the observation column.

```ruby Factor Reduction
def reduce!(evidence, norma=true)
  @vars.each_with_index do |rv, i|
    if evidence.has_key?(rv)
      rv.ass.each_with_index do |as, j|
        if as!=evidence[rv]
          temp = [true]*@vars.size
          temp[i] = j
          @vals[*temp] = 0.0
        end
      end
    end
  end
  normalize!() if norma
  self
end
```

## Multiplication

The key method. Given two factors I modify each one by:

- gathering all the variables of the resulting multiplication factor (union of all sorted variables)
- inserts new variables/axis in each factor so they have the same axes (simple rotations)
- we expand the values ndarray on each new axis by repeating a number of times equal to the cardinality of the axi's variable.

Continuing with the graphic example, to expand our previous factor (variables v1 and v2) by another variable (v3) we would start with the 2D values along axes v1, v2. Then we add a third dimension for v3.

{% img center /images/nov13/multiply1.png %}

And then we repeat the 2D matrix (v1,v2) along the v3 axis. In our case v1 and v2 have cardinality 2 and v3 has cardinality 3 so we repeat the 2D matrix twice more along the v3 axis.

{% img center /images/nov13/multiply2.png %}

At the end of the process we have two NArrays that represent the same variables, aligned and of the same shape. To multiply we only need to multiply element wise. At the end of the day, this Ruby method is 30% smaller and yet faster than the python version.

```ruby Factor Multiplication

def multiply!(other, norma=true)
  return self unless other
  all_vars = [*@vars, *other.vars].uniq.sort

  narr1, narr2 = self.expand_axes(all_vars), other.expand_axes(all_vars)
  na = narr1*narr2
  suma  = na.sum

  @vars = all_vars
  @cards = @vars.map{|e| e.card}
  @vals = na.reshape!(*@cards)

  normalize!(suma) if norma
  self
end

def expand_axes(whole_vars)
  return @vals.flatten if @vars==whole_vars

  multiplier = 1.0
  new_vars = whole_vars.reject{|rv| @vars.include?(rv)}

  old_order_vars = [*@vars, *new_vars]
  new_order = whole_vars.map{|e| old_order_vars.index(e)}

  new_cards = new_vars.map{|v| multiplier *= v.card; v.card}

  flat = [@vals.flatten] * multiplier
  na = NArray.to_na(flat).reshape!(*@cards, *new_cards)

  if new_order != [*(0..whole_vars.size)]
    na = na.transpose(*new_order)
  end

  return na.flatten
end
```

