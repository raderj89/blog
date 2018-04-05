---
layout: default
title: "Verbal Math in JavaScript"
date: 2014-10-04 09:33
comments: true
categories: technical
tags: [javascript, functional programming]
---

I'm a fan of this fluent calculator exercise (I covered my solution using Ruby in [my last blog post](http://www.jaredrader.com/blog/2014/09/24/verbal-math-in-ruby/)), and on another flight, I decided to implement a solution in JavaScript. Ruby's object-oriented nature and metaprogramming capabilities made the task fairly simple; how would it go in JavaScript?

Recall that this exercise requires one to be able to verbally enter math operations, i.e. `one(plus(one()))`, and return the correct integer result.

In Ruby, because most everything is an object, I was able to call methods on my integers, like `1.send(:+, 1)`. In JavaScript, integers are just primitive data types - not objects that can receive messages. But JavaScript's ability to return functions allows us to achieve the same ability.

Again, I started off with the `one()` function. I knew `one()` would need to optionally take an argument, which would be an operator function. If no operator function was passed, I simply return 1. Otherwise, I need to return an operation on 1:

```javascript
function one(opFunc) {
  if (opFunc == undefined) {
    return 1;
  } else {
    // as yet unwritten code
  }
}
```

<!-- more -->

In the `else` branch, I know I need to do something with `one`'s value. Like I said earlier, I can't call methods on 1, so it seemed like my only option was to pass 1 to a function.

But how do I pass 1 to a function when the function I'm passing into `one()` already takes an argument?

A review of Eloquent JavaScript by Marijn Haverbeke helped me find the right answer. In the chapter on functions, Haverbeke says:

>"you should consider functions to not just package up a computation, **but also an environment**. Top-level functions simply execute in the top-level environment, that much is obvious. But a function defined inside another function **retains access to the environment that existed in that function at the point when it was defined.**"

He illustrates this explanation with a function that I was ultimately able to use as my `plus()` function:

```javascript
function makeAddFunction(amount) {
  function add(number) {
    return number + amount;
  }
  return add;
}

var addTwo = makeAddFunction(2);
var addFive = makeAddFunction(5);
show(addTwo(1) + addFive(1));
//=> 9
```

I had to stare at this for a while before finally wrapping my head around it. `makeAddFunction()` takes an `amount`, defines a function within itself called `add()`, which takes a `number`, and returns the sum of `number` and `amount`. `add`, the function, is returned.

`var addTwo` is assigned to `makeAddFunction(2)`. If you run `console.log(addTwo)`, you'll see that `addTwo`'s value comes back as `[Function: add]`. What may not be immediately apparent is that this function has stored `amount`. `makeAddFunction(2)` defines `add()`, so `add()` retains `amount`'s value, which is 2.

Because `addTwo` is now set to a function that expects an argument, you can call `addTwo(1)`, which passes 1 to `add()`, which returns 3 (1 plus `amount`, which was set to 2 when `makeAddFunction(2)` was called).

That's a lot of whiches.

So I knew in my program, the `plus(one())` passed to `one()` returns a function `add`, and within `add`'s environment, the `amount` is set to 1. I can then call `add()` within my `one()` function, passing `add()` the integer 1. The result looks like this:

```javascript
function one(opFunc) {
  if (opFunc == undefined) {
    return 1;
  } else {
    return opFunc(1);
  }
}

function plus(amount) {
  function add(number) {
    return number + amount;
  }
  return add;
}

console.log(one(plus(one())));
//=> 2
```

Now I can define the same operations for numbers 1-9 and create the rest of the operation functions needed to subtract, multiply and divide numbers.
