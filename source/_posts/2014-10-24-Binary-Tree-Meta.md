---
layout: post
title: "Ruby Metaprogramming II"
date: 2014-10-24 8:01
comments: true
categories: Ruby, Metaprogramming
---

This is a follow up to my path of learning Ruby Metaprogramming. It follows my previous post about the [Equalizer gem](/blog/2014/10/05/Metaprogramming/).

For this exercise we are going to start with a basic BinaryTree that consists of nodes that just hold a key (val), and two links to the left and right children. This is not a BST, just a raw binary tree where each node is considered a tree in itself.

```ruby
class BinaryTree
  attr_accessor :val, :left, :right

  def initialize(val)
    @val = val
    @left = @right = nil
  end
end
```

For our example we are going to create an ad-hoc tree that we wire in the following manner.

{% img center /images/Oct14/binary_tree.png 300 %}

<!--more-->

```ruby
tim    = BinaryTree.new('Tim')
phil   = BinaryTree.new('Phil')
jony   = BinaryTree.new('Jony')
dan    = BinaryTree.new('Dan')
katie  = BinaryTree.new('Katie')
craig  = BinaryTree.new('Craig')
eddie  = BinaryTree.new('Eddie')
peter  = BinaryTree.new('Peter')
andrea = BinaryTree.new('Andrea')

tim.left, tim.right = jony, phil
jony.left, jony.right = dan, katie
katie.left, katie.right = peter, andrea
phil.left, phil.right = craig, eddie
```

For this exercsie we are only interested in traversing the tree and outputting to stdout the names of each node. There are three different ways to perform this traversal in a depth-first fashion: pre-order, in-order and post-order. You can read the details about each one in this [Wikipedia page](http://en.wikipedia.org/wiki/Tree_traversal).

After a bit of reflection and sandboxing, a good and common-sense way to implement becomes clear:

```ruby
def traverse_pre_order
  puts val
  left.traverse_pre_order if left
  right.traverse_pre_order if right
end

def traverse_in_order
  left.traverse_in_order if left
  puts val
  right.traverse_in_order if right
end

def traverse_post_order
  left.traverse_post_order if left
  right.traverse_post_order if right
  puts val
end
```

The code is clear and readable, yet suffers from some repetition. We also see a pattern emerging: in all cases we recursively call the traverse method first on the left and then on the right, and the position of the call to stdout depends on the specific flavor of traversal:

- pre => first call
- in  => between left and right
- post => after both calls

This is a great opportunity to tinker with metaprogramming to see if we can do better. I focus on leveraging Ruby's define_method, which allows us to create multiple methods with a single definition.

```ruby
%w[pre in post].each do |prefix|
  define_method("traverse_#{prefix}_order") do
    # statements_to_execute = [traverse_left, traverse_right]
    # insert the puts call in the right place depending on the prefix
    # execute the array of orders one by one
  end
end
```

The idea is to have an array made of little bunches of code to execute. Then we can manipulate the order in which those code snippets are executed by re-ordering the array. The array becomes something like an instructions book, an ordered set of steps to perform.

To insert the chunks of code in the array, we need to package them into objects. A solution is to wrap them as blocks of functionality as Proc objects (I have used stabby lambdas). To execute one of these object we just call it to yield.

```ruby
instructions = [
  -> { left.traverse_pre_order if left },
  -> { right.traverse_pre_order if right }
]
```

Now, is only a question of changing the order of the array of statements and adapt it to each flavor of traversal. Once the array is reshuffled to our taste, we iterate over each cell, anc execute the proc.

Here is the final code of the complete class unrefactored. I have taken advantage of the yet to be defined method inside, it cannot get more dynamic than that!

```ruby
class BinaryTree
  attr_accessor :val, :left, :right

  def initialize(val)
    @val = val
    @left = @right = nil
  end

  %w[pre in post].each_with_index do |prefix, index|
    define_method("traverse_#{prefix}_order") do
      instructions = [
        -> { left.public_send("traverse_#{prefix}_order") if left },
        -> { right.public_send("traverse_#{prefix}_order") if right }]
      do_stuff = -> { puts val }
      instructions.insert(index, do_stuff)
      instructions.each { |x| x.yield }
    end
  end
end
```

Specially interesting is how the scoping works, which I'll leave for a future post. Stay tunned!

Here is the final version, refactored, plus some tests.

```ruby
class BinaryTree
  attr_accessor :val, :left, :right

  def initialize(val)
    @val = val
    @left = @right = nil
  end

  %w[pre in post].each_with_index do |prefix, index|
    define_method("traverse_#{prefix}_order") do
      instructions(prefix).insert(index, do_stuff).each(&:yield)
    end
  end

  private

  def do_stuff
    -> { puts val }
  end

  def instructions(prefix)
    [left, right].map { |e| -> { e.send("traverse_#{prefix}_order") if e } }
  end
end
```

```ruby
require 'spec_helper'

describe BinaryTree do
  describe 'Search Methods' do
    let(:bt) { # tim as defined above }
    it 'BinaryTree#traverse_pre_order' do
      proc { bt.traverse_pre_order }.must_output "Tim\nJony\nDan\nKatie\nPeter\nAndrea\nPhil\nCraig\nEddie\n"
    end

    it 'BinaryTree#traverse_in_order' do
      proc { bt.traverse_in_order }.must_output "Dan\nJony\nPeter\nKatie\nAndrea\nTim\nCraig\nPhil\nEddie\n"
    end

    it 'BinaryTree#traverse_post_order' do
      proc { bt.traverse_post_order }.must_output "Dan\nPeter\nAndrea\nKatie\nJony\nCraig\nEddie\nPhil\nTim\n"
    end
  end
end
```
