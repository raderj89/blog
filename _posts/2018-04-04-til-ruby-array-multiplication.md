---
layout: default
title: TIL - Ruby Array Multiplication
date: 2018-04-04 21:25 -0700
categories: [til, technical]
tags: [til, ruby]
---

Today I learned that if you call a multiplication operator on a Ruby array, it duplicates each element of the array by the multiplier.

So for example:

```shell
>> [1,2,3] * 3
=> [1, 2, 3, 1, 2, 3, 1, 2, 3]

>> ['Jared Rader'] * 3
=> ["Jared Rader", "Jared Rader", "Jared Rader"]
```
<!-- more -->

None of the other math operators work, with the exception of `+` - though this just performs array concatenation with the right-hand side of the operation needing to be an array.

I'm not really sure when I'd use such an operation. I saw a beginner using it for a solution to a code challenge in which they were given a sentence and a name and had to replace the name in the sentence with X's, and the number of X's had to be twice as long as the name's length. So the beginner did something like:

```ruby
name_array = name.split('') * 2

x_name = []
name_array.each { |_| x_name << 'X' }
```

It wasn't the most elegant thing in the world, and I'm not even sure this person knew exactly what they were doing, but using this quirky feature, he solved his challenge.

I half expected `**` to work, but alas, it did not. But we can write one ourselves:

```ruby
class Array
  def **(num)
    raise ArgumentError, 'negative argument' if num < 0

    return self if self.length == 1
    return self.uniq if num.zero?

    self * (self.length ** num)
  end
end
```

This allows me to write `['Jared', 'Rader'] ** 3` and will give me the same result as `['Jared' 'Rader'] * 8`. There are two elements, so 2<sup>3</sup> means we want eight elements for each element.

Also, because anything to the zero power is 1, I figured this would mean we simply want one of each element in the array, so I call `self.uniq`.

Again, probably useless, but a fun little hack all the same.
