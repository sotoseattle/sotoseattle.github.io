---
layout: post
title: "PRIMO: Random Variables"
date: 2014-07-16 8:01
comments: true
categories: PGN, Ruby, Primo
published: false
---

## Alás Primo!

I have decided to code all this in Ruby instead of Python. The reason being that Ruby is more flexible in order to build prototypes that encompass all aspects of the functionality. The apps that I would like to build around ML are like cars. The engine may be a key component and its performance paramount, but there is more to a car than its engine. Our apps will be more than its inference engine, and in all those other aspects Ruby shines. If I was obsessed only with the engine I would have code it all in Julia or C, but I am in this for the fun and the possibilities, and Ruby is a pleasure to play with.

Besides, what I have already coded in Ruby is already faster than my Python code (which shows what a beginner I am in Python). The key to the performance boost has been the use of the [NArray gem from Masahiro Tanaka](http://masa16.github.io/narray/), which allows me to, for example, multiply two multidimensional arrays element wise in a single step, after aligning them with simple rotations of their axes (actually pretty cool).

I have christened this working library as PRIMO (Probabilistic Inference Modeling), and the code is available at https://github.com/sotoseattle/PRIMO

It works along the same lines as the Python version so the following posts will be a bit repetitive since we are rehearsing much of what was done months ago in Python. This is an ongoing effort that I hope to culminate with a working CRF application that estimates its own parameters.

## Random Variables

The essential building block of Primo, similar to Nodes in graphs, each holds the following instance variables:

- **cardinality**. For example, a binary variable that can only take two values (true-false, 0-1) and therefore it has a cardinality of 2. The roll of a dice would have cardinality of 6, because the the outcome can take 6 different values: 1, 2, 3, 4, 5 or 6.
- **ass** (assignments). An ordered array with all the possible assignments. If no assignments are given it uses integers starting from 0. The previous dice roll would have an ass of [0, 1, 2, 3, 4, 5]; a binary variable would have an ass of [0, 1]. We can also make a binary random variable for the health of a patient with the assignment array ['healthy', 'sick'].
- **name**. Just a name to make it easier to identify the variable.

An important detail is how we define the <=> operator because we will be comparing between random variables based on their internal object ids. This is necessary because later on, when we multiply sets of variables, we will do it according to their order. The order itself doesn't matter, all we'll need is that there is a way to order them in a stable, immutable and persistent way.

```ruby
  def initialize(args)
    args.merge(name: '', ass: nil)

    @card = args[:card].to_i
    @name = args[:name].to_s
    @ass = args[:ass] ? Array(args[:ass]) : [*0...@card]

    fail ArgumentError if @card == 0 || @ass.size != @card
  end

  def <=>(other)
    object_id <=> other.object_id
  end

  def [](assignment)
    ass.index(assignment)
  end

  def to_s
    "#{name}"
  end
end
```
