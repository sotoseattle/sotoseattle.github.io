---
layout: post
title: "Ruby Continuations"
date: 2015-12-05 12:14:40 -0800
comments: true
categories: Ruby
---

Continuations are a useful device for many coding tasks. I like to visualize them as portals or wormholes that connect different parts of your code. Continuations allow you to hyper-jump from one place of your executing code to another without missing a bit. The following is an example of how they can be used:

{% img center /images/dec15/portal.jpg 800 %}
<!--more-->

> Given a binary tree and two nodes A and B from within that tree, write a function to find the node which is the first common ancestor of A and B.

The first thing to do is find the chain of parents of a given node inside a tree. Once we have the genealogy of both nodes (A & B), it is fairly simply to find the common ancestor. It is in this method for finding the parents of a node where we use continuations.

You already know how to search binary trees, depth or breadth first, and you could just traverse the tree in search of the nodes and pick up the parents as you go along. That is not intrinsically difficult and there are many examples on the Internet. The problem is that you need to run through the whole tree every time you initiate a search. Could there be a way to stop moving along the tree once you find the node and simple return to sender? Yes, with continuations.

{% img right /images/dec15/treesearch.png 300 %}

Intuitively what we want is something similar to the following:

- We open a portal at the root or beginning of the tree. This portal is where we want to go back to when we find The Node. It is our way back home for when we get lost in the recurring calls throughout the tree.
- Once we secure a way back, we jump into the rabbit hole, we start searching the tree, going from node to node, a dazzling pin-ball.
- As we go along, as we search for The Node, we keep a bread crumb track of the path taken.
- After many jumps among nodes, already disoriented, we arrive to The Node we were after. But, alas! our tree is made of nodes that do not link upwards to parents, only downwards to children, so we cannot use our bread crumbs to dig ourselves back to the root!
- No sweat, we have an opened portal at the root and, as we all know from Cosmology 101, we can always open the other end of the wormhole with our sonic screwdriver and get back to safety.

The final code would look like the following. I leave up to you to make sense of the details (the Internet is full of better explanations than this one :-).

```ruby
module Ancestry
  require 'continuation'

  def ancestors node
    portal = callcc {|cont| cont}                   # open cosmological-portal (brrrrzzzzzuinga!!)

    return portal unless portal.is_a? Continuation  # return whatever was sent through the portal

    find node, [], portal                           # go down the rabbit hole in search of node
  end

  def find n, crumbtrail, getaway
    getaway.call crumbtrail if self == n            # if found, send ancestry logbook through the portal

    (@left  && @left.find( n, crumbtrail + [self], getaway)) ||
    (@right && @right.find(n, crumbtrail + [self], getaway))
  end

  def common_ancestor a, b
    (ancestors(a) & ancestors(b)).last
  end
end

class Bintree
  include Ancestry
  attr_accessor :val, :left, :right

  def initialize val
    @left = @right = nil
    @val = val
  end
end
```
