---
layout: post
title: "Ruby Metaprogramming in Action"
date: 2014-10-05 8:01
comments: true
categories: Ruby, CodeFellows, Metaprogramming
---

The [Equalizer gem](github.com/dkubb/equalizer) provides a nifty example of Ruby metaprogramming.

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

## Include instance of a Module?

```ruby
# example
class GeoLocation
  include Equalizer.new(:latitude, :longitude)
...
```

The first thing that kicks your gut when reading the code is the instantiation of the module. What? Weren't modules used to be abstract, nice little packages of functionality, classes devoid of the perils and tribulations born out of fleshed-out Objects?

In Ruby everything is an Object, as we wouldn't deny the right to be objectified to anybody, so the following two ways of coding a module are equivalent:

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

Now that we see that everything is an object we inspect Equalizer's code:

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

Aha! It passes certain attributes of the GeoLocation class (passed as keys: latitude and longitude) to make them comparable. For this we need to customize the methods inside the module (because different objects will have different attributes, and we want something reusable).

And there lies the rub, the double nature of Equalizer.

1. Equalizer is a Module and by 'including' it, GeoLocation makes all methods from Equalizer available to his objects.
2. Yet Equalizer is defined as a class that can be instantiated.

But we have seen that everything is an object, so the instantiation must serve a different purpose.

Yes, by defining Equalizer as a class that can be instantiated, at that moment of instantiation we can pass the specific instance variables to customize those methods (the 'keys'). So Equalizer defines methods based on something undefined (keys), and only at inclusion (when we know which are the instance variables to make comparable) it customizes its module's methods on the fly.

Although a stateless, Equalizer is able to be adapt to the circumstances of each class that will include it. In our example, at instantiation it uses GeoLocation's latitude and longitude to redefine its methods at the last minute, to the effect of adding to them the ability to be comparable.

Let's check it works:

```ruby
class Insane < Module
  def initialize(key)
    define_method(key) { "Oh My God!" }
  end
end

class Node
  include Insane.new(:val)
end

a = Node.new
puts "a.val = #{a.val}"
=> a.val = Oh My God!
```

Convoluted, pirouetticall, and yet it works. We can define an instance variable through an instantiated module included at runtime. Let's continue before we finish the stock of Tylenol.

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
