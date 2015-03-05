---
layout: post
title: "Seattle Ruby Game Of Life"
date: 2015-03-04 10:00
comments: true
categories: Ruby
---

{% img right /images/mar15/wheel.png 250 %}

Last Tuesday the Seattle Ruby group organized a workshop to code Conway's Game of Life. It was a first (at least in a long time), and I think it was a success. People were engaged and everybody was focused coding away in the 45 minutes we had. We even had a Wheel of Misfortune so people could add randomized constraints in case the problem was too easy.

A repository was published prior to the event so people could familiarized themselves with the problem. [Repo](https://github.com/SeaRbSg/workshops). The following are some examples included in the common repo from some contributors:

Lito Nicolai had an amazing implementation using matrices:

<!--more-->

<br/>

```ruby
require 'matrix'
require './matrix_math'

def sum l
  l.reduce :+
end

def twos grid
  grid.map{|i| if i == 2 then 1 else 0 end}
end

def threes grid
  grid.map{|i| if i == 3 then 1 else 0 end}
end

def neighbors grid
  sum( [-1, 0, 1].product([-1, 0, 1]).map{|x, y| grid.rotate x, y } ) - grid
end

def life grid
  ((twos neighbors grid) & grid) | (threes neighbors grid)
end
```


For my part, the most simple code I could come up with was a purely functional implementation:

```ruby
module GOL
  def tick(living)
    potential = living.map { |c| neighbors(c) }.flatten(1).uniq - living

    new_board = living.select { |cell| alive_around(cell, living).between?(2, 3) }
    new_board += potential.select { |cell| alive_around(cell, living) == 3 }

    return new_board
  end

  def neighbors(cell)
    x, y = cell
    [x + 1, x, x - 1].product([y - 1, y, y + 1]) - [[x, y]]
  end

  def alive_around(cell, board)
    neighbors(cell).count { |c| board.include? c }
  end
end
```

Ryan Davis not only contributed with code but with a visualization library based on GOSU. See it in action as it helps visualize the code of two different approaches written by Ryan and Scott Windsor.

<br/>

{% img left /images/mar15/zen_gol.gif 420 %}
{% img right /images/mar15/sentientmonkey_gol.gif 420 %}

