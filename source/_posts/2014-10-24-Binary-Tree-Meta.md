---
layout: post
title: "Ruby Metaprogramming and Trees"
date: 2014-10-24 8:01
comments: true
categories: Ruby, Metaprogramming
---

Learning the ropes of Ruby Metaprogramming, here is small experiment I coded.

I start with a basic BinaryTree that consists of basic nodes that hold a key (val), and two links to the left and right children. This is not a BST, just a raw binary tree where each node is considered a tree in itself.

```ruby
class BinaryTree
  attr_accessor :val, :left, :right

  def initialize(val)
    @val = val
    @left = @right = nil
  end
end
```

Now we instantiate the tree and wire it.

{% img center /images/Oct14/binary_tree.png 400 %}

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

We are only interested in traversing the tree and outputting to stdout the names of each node. There are three different ways to perform this traversal in a depth-first fashion: pre-order, in-order and post-order. You can read more about them in this [Wikipedia page](http://en.wikipedia.org/wiki/Tree_traversal).

A common sense way to implement it is the following:

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

The code is clear and readable, yet suffers from some repetition. Also we can see a pattern emerging: in all cases we recursively call the traverse method first on the left and then on the right, and the position of the call to stdout depends on the specific flavor of traversal:

- pre => first call
- in  => between left and right
- post => after both calls

We can do better with metaprogramming by leveraging Ruby's define_method:

```ruby
%w[pre in post].each do |prefix|
  define_method("traverse_#{prefix}_order") do
    # statements_to_execute = [traverse_left, traverse_right]
    # insert the puts call in the right place depending on the prefix
    # execute the array of orders one by one
  end
end
```

Since we need to manipulate the order of the statements to execute, we need to make them objects, so we wrapp them in blocks of functionality (stabby lambdas). Then, is only a question of changing the order of the array of statements to adapt it to each flavor of traversal, and at the end we call each statement to yield. Here is the final code of the complete class.

```ruby
class BinaryTree
  attr_accessor :val, :left, :right

  def initialize(val)
    @val = val
    @left = @right = nil
  end

  %w[pre in post].each_with_index do |prefix, index|
    define_method("traverse_#{prefix}_order") do
      left_to_right = [
        -> { left.public_send("traverse_#{prefix}_order") if left },
        -> { right.public_send("traverse_#{prefix}_order") if right }]
      do_stuff = -> { puts val }
      left_to_right.insert(index, do_stuff)
      left_to_right.each { |x| x.yield }
    end
  end
end
```

Specially interesting is how the scoping works. But I am going to leave it for a future post. Stay tunned!

This could work for testing purposes:

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
