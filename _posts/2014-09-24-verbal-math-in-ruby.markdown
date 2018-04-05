---
layout: default
title: "Verbal Math In Ruby"
date: 2014-09-24 13:15
comments: true
categories: technical
tags: ruby
---

Came across an interesting code challenge on Code Wars a while back. It involved defining methods in a way to be able to type mathematical operations verbally, i.e. `one plus one` which would return `2`.

I pondered over this for a while, and on a flight back from San Francisco came up with a neat solution and even learned a couple interesting things about metaprogramming.

I started with defining `one`. I know that I'll want to potentially pass an argument to `one`. If I don't have an argument, I'll just want to return the integer `1`. Simple enough. I can just give my method definition argument a default value of `nil`, check whether I have an operator, and if I don't, simply return the integer `1`:

```ruby
def one(operator_method = nil)
  if operator_method
    # as yet undefined code
  else
    1
  end
end
```

Ok, so what should happen when I get an operator? I decided to find a way to implement `plus`.

<!-- more -->

This method is a bit tricky - for it to work, it needs to take an argument that is one of my number methods. And it needs to return a value in such a way that the method taking this operator can do something with the return value.

This brought me back to the if statement in `one`. I could write what I needed the number method to do with the expected result of the operator method.

I knew I wouldn't be able to finagle the `+` method in there. Instead, I decided to play around with `.send()`. You can call `.send()` on any object, passing it a method to call on the receiver, and an argument to supply to the method. For example `1.send(:+, 1)` returns `2`. Ruby knows to convert a symbol to a method, so this is basically equivalent to writing `1.+(1)`.

This gave me the answer to what I needed each of my word operator methods to return: Their appropriate operators, and then the return value of the number method passed to the word operator method.

So `plus` became:

```ruby
def plus(number_method)
  return :+, number_method
end
```

This returns `[:+, 1]` if I were to call `plus(one)`. Now I have my operator and number that I can assign to variables using Ruby's multiple assignment feature:

```ruby
def one(operator_method = nil)
  if operator_method
    number, operator = operator_method
    1.send(operator, number)
  else
    1
  end
end
```

So that's cool. I began defining the same methods for the rest of the numbers to nine. I was duplicating a lot of code - the only thing different about each method was either the operator or the specific number. Wouldn't it be great if I could define the methods on the fly?

Well that's easy with Ruby's metaprogramming capability. To begin, I created two constant hash values, `NUMBERS` and `OPERATORS`:

```ruby
OPERATORS = {
  plus:       :+,
  minus:      :-,
  times:      :*,
  divided_by: :/
}

NUMBERS = {
  one:   1,
  two:   2,
  three: 3,
  four:  4,
  five:  5,
  six:   6,
  seven: 7,
  eight: 8,
  nine:  9
}
```

I wanted to use Ruby's `define_method()` method, but I was a little confused about a couple things. Specifically, I didn't know how to define a method that accepted an argument or an argument with a default value. However, with some tinkering, I managed to get it working. Here's what the solution looked like:

```ruby
OPERATORS.each do |word, op|
  define_method(word) do |number|
    return op, number
  end
end

NUMBERS.each do |word, num|
  define_method(word) do |word_operator = nil|
    if word_operator
      operator, number = word_operator
      num.send(operator, number)
    else
      num
    end
  end
end
```

It seems the placeholder variable passed to the code block for `define_method()` becomes the argument that gets passed to the method being defined. And then, I didn't realize you could do this, I can even give the placeholder a default value, as you see in the `NUMBERS` loop: `|word_operator = nil|`. This successfully defines the methods needed, so now I can add, subtract and divide to my heart's content.

```ruby
puts one
#=> 1
puts one minus one
#=> 0
puts two times three
#=> 6
puts nine divided_by three
#=> 3
```

One thing I noticed is that this solution doesn't work if you want to chain operations, like `two times three plus six`. The order of operations is ignored because the innermost methods are evaluated first. So `two times three plus six` returns `18` instead of `12` as would be expected if you followed the order of operations. It does work for chaining addition operations, because it doesn't matter in which order the numbers are added. It'd be interesting to know if there's a way to accomplish making the methods aware of order of operations.
