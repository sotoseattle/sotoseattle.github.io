---
layout: post
title: "Ruby Metaprogramming in Action"
date: 2014-10-05 8:01
comments: true
categories: Ruby, CodeFellows, Metaprogramming
---

The [Equalizer gem](https://github.com/dkubb/equalizer) provides a nifty example of Ruby metaprogramming.

It is a module that when added to your class helps define equality, equivalence and inspection methods.

```ruby
class GeoLocation
  include Equalizer.new(:latitude, :longitude)

  attr_reader :latitude, :longitude

  def initialize(latitude, longitude)
    @latitude, @longitude = latitude, longitude
  end
end

point_a = GeoLocation.new(1, 2)
point_b = GeoLocation.new(1, 2)

point_a.inspect    # => "#<GeoLocation latitude=1 longitude=2>"

point_a == point_b           # => true
point_a.hash == point_b.hash # => true
```
<!--more-->

## Include instance of a Module?

```ruby
# example
class GeoLocation
  include Equalizer.new(:latitude, :longitude)
...
```

The first thing that calls the attention when reading the code is the instantiation of the module. What? Weren't modules used to be abstract, nice little packages of functionality, angelical and stateless, devoid of the tribulations of commoner fleshed-out Objects?

In Ruby everything is an Object, and the following two ways of coding a module are equivalent:

```ruby
module Insane
  def hello
    'hola'
  end
end

Insane = Module.new do
  def hello
    'hola'
  end
end
```

Let's inspect the Equalizer's code:

```ruby
# equalizer.rb
class Equalizer < Module
  def initialize(*keys)
    @keys = keys
    define_methods
    freeze
  end
  ...
end

# example
class GeoLocation
  include Equalizer.new(:latitude, :longitude)
  ...
```

In the last line lies the rub, the double nature of Equalizer.

1. Equalizer is a Module and by 'including' it, GeoLocation makes all methods from Equalizer available to his objects.
2. Yet Equalizer is defined as a class that can be instantiated.

But we have seen that everything is an object and there is no need to make it a class object, so the instantiation must serve a different purpose.

Yes, by defining Equalizer as a class that can be instantiated, at that moment of instantiation we can pass the specific instance variables in an instant (sorry!) to customize those methods based on the keys passed (latitude and longitude). So Equalizer defines methods based on something undefined (keys), and only at inclusion (when we know which are the instance variables to make comparable) it customizes its module's methods on the fly.

Although stateless, Equalizer is able to be adapt to the circumstances of each class that will include it. In our example, at instantiation it uses GeoLocation's latitude and longitude to redefine its methods at the last minute, to the effect of adding to them the ability to be comparable.

Let's check it works:

```ruby
class Insane < Module
  def initialize(key)
    define_method("#{key}?") { 'God!' }
  end
end

class Person
  attr_reader :who_am_i
  include Insane.new(:who_am_i)

  def initialize(name)
    @who_am_i = name
  end
end

a = Person.new('javier')
puts "a.who_am_i = #{a.who_am_i}"
=> a.who_am_i = javier
puts "a.who_am_i? = #{a.who_am_i?}"
=> a.who_am_i? = God!
```

Convoluted, pirouettical, and yet it works.

## Define Method

Among other metaprogramming tricks it uses the define_method extensively. Although this is usually done to create a new named method at runtime, in this case, the method's names are set from the start, what is created dynamically is the way the method operates.

For example, the cmp? (comparable?) method has a set name (cmp?) and the blocks passed is also well defined (we check that all keys of both objects return the same values), but the fact that we won't know which keys are available to compare until runtime makes this use an example of metaprogramming.

```ruby
# where we make attributes comparable
def define_cmp_method
  keys = @keys
  define_method(:cmp?) do |comparator, other|
    keys.all? { |key| send(key).send(comparator, other.send(key)) }
  end
  private :cmp?
end

# The comparisson method
def ==(other)
  other = coerce(other) if respond_to?(:coerce, true)
  other.is_a?(self.class) && cmp?(__method__, other)
end
```

Two notes:

1. The cmp? method is just a DRY method used to define the real comparison methods (eql? and ==). This way it is easy to extend.

2. Consider also that we include this Equalizer class as an instantiated object when defining the class, so when we instantiate a new object all Equalizer methods are available as instance methods already defined for the existing instance variables.
