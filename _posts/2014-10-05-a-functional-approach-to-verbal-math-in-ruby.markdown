---
layout: default
title: "A Functional Approach to Verbal Math in Ruby"
date: 2014-10-05 11:12
comments: true
categories: technical
tags: [ruby, functional programming, metaprogramming]
---

Inspired by [my functional approach](http://http://www.jaredrader.com/blog/2014/10/04/verbal-math-in-javascript/) to solving the verbal math problem in JavaScript, I decided to implement a similar approach in Ruby.

My [first solution](http://www.jaredrader.com/blog/2014/09/24/verbal-math-in-ruby/) took advantage of Ruby's everything-is-an-object nature, sending number objects symbols and values, which made solving the problem pretty easy.

However, it's possible to take a functional approach in Ruby that is nearly equivalent to how you could solve this problem in JavaScript. And then, you can even combine this with Ruby's object-oriented nature and metaprogramming capabilities to create a truly elegant solution.

While JavaScript has anonymous functions, Ruby has something similar in blocks, procs, and lambdas.

Here's a language conversion to illustrate how you can achieve very similar functionality in Ruby and JavaScript (Check out [my last post](http://http://www.jaredrader.com/blog/2014/10/04/verbal-math-in-javascript/) where I explain the JavaScript function in-depth):

JavaScript:

```javascript
function plus(amount) {
  function add(number) {
    return number + amount;
  }
  return add;
}
```
<!-- more -->

Ruby:

```ruby
def plus(amount)
  add = Proc.new { |num| num + amount }
  return add
end
```

How cool is that? My `plus()` method defines a Proc called `add` that takes `num` and adds it to `amount`. That proc is what's ultimately returned.

I overengineered the Ruby implementation just a bit to make it look as close to the JavaScript `plus()` function as much as possible, but Ruby allows me to be even more succinct - I could have left out creating the `add` variable and simply returned `Proc.new { |num| num + amount }`.

Now check out the number functions side-by-side:

JavaScript:

```javascript
function one(opFunc) {
  return opFunc ? opFunc(1) : 1;
}
```

Ruby:

```ruby
def one(op_method = nil)
  op_method ? op_method.call(1) : 1
end
```

They operate pretty much the same. In Ruby now, the `plus(one)` inside of a simple `two(plus(one))` call evaluates to a proc where `amount` equals 1. So when it's passed to `two()`, you get `add.call(2)`, which passes 2 as `num` and adds it to `amount` which is set when passing `one` to `plus`.

I can now combine this functional approach with Ruby's class-based data types and metaprogramming to create a very cool solution:

```ruby
%w(zero one two three four five six seven eight nine).each_with_index do |num, index|
  define_method(num) do |op_method = nil|
    op_method ? op_method.call(index) : index
  end
end

%w(plus + minus - times * divided_by /).each_slice(2) do |word_op|
  word, op = word_op

  define_method(word) do |amount|
    Proc.new { |num| num.send(op, amount * 1.0) }
  end
end

one plus one
#=> 2.0
nine divided_by four
#=> 2.25
eight times 4
#=> 32.0
```

Pretty cool, huh? Notice that I'm multiplying every `amount` by 1.0 so as to avoid `ZeroDivisionError`s.

For a great article comparing JavaScript functions and Ruby blocks, procs and lambdas, check out [Skilldrick's blog](http://skilldrick.co.uk/2011/01/ruby-vs-javascript-functions-procs-blocks-and-lambdas/).
