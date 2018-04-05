---
layout: default
title: "Thoughts on Ruby Inheritance Patterns"
date: 2016-08-30 20:40
comments: true
categories: technical
tags: [ruby, rails, object oriented programming]
---

Developers writing code in object-oriented languages like Ruby and JavaScript have a complicated relationship with inheritance. I have seen opinions stating that class inheritance is absolutely evil and should be avoided in all circumstances, and those who favor composition over inheritance, but don't discount class inheritance entirely.

Those that are against class inheritance entirely often favor composing classes with modules to add behavior. The funny thing here is that including modules functions pretty much exactly the same as class inheritance. If a method is not defined in the class, Ruby looks up the inheritance tree, starting with the modules included.

The favor for composition comes from the fact that composing classes with modules is more flexible than what a class hierarchy offers and classes can essentially achieve multiple inheritance through module inclusion.

I decided to take the idea to its extreme recently in a project at work. After students enroll in courses at Bloc, they are taken through an onboarding workflow where they create a profile, select their mentor, create a schedule of appointments, etc. We created a neat class called `OnboardWorkflow` that defined a series of determined steps so that students would go through onboarding in the order we wanted them to.

<!-- more -->

Recently, we had to create another workflow for students to go through after enrollment - filling out legal forms now required by the California Bureau of Postsecondary Private Education (BPPE). Students would need to fill out 3 or 4 forms before we could process payment.

Knowing we already had an `OnboardWorkflow` class, I started adding more steps to include the enrollment form flow. But eventually, I realized the two workflows, while very similar, really ought to be separate from each other.

I thought about creating a base `Workflow` class that includes the common class and instance methods and then subclassing the distinct workflows. This would look something like:

```ruby
# workflow.rb
class Workflow
end

# onboard_workflow.rb
class OnboardWorkflow < Workflow
end
```

But I recalled all the hate and fear around inheritance, so I decided to try composition instead.

I created a `Workflow` module that carries the common behavior. It packages the class and instance methods, using the `self.included` callback so that I can properly `extend` the class methods and `include` the instance methods:

```ruby
module Workflow
  def self.included(base)
    base.extend ClassMethods
    base.send :include, InstanceMethods
  end

  module ClassMethods
    # ...
  end

  module InstanceMethods
    # ...
  end
end
```

And then in the class:

```ruby
class EnrollmentFormsWorkflow
  include Workflow
end
```

This works as expected, but at the same time, I'm not completely convinced that this is a big improvement over simply having a base `Workflow` class. For one, this module is very specific; it can't be included into classes other than workflow-specific ones. Modules really shine when they hold behavior that can be shared across many different classes. For two, my subclasses can be accurately described as workflows, so the 'is-a' test passes: an `OnboardWorkflow` is a `Workflow`.

A good example in the Bloc code base is our `Versionable` module. We use it to make curriculum content easily versionable. A good hint that you might be creating behavior that's good for modules is if you can describe it in that '-able' term: version-able, read-able, etc. Saying something is workflow-able sounds pretty awkward. And I think that's because a workflow is its own domain, as opposed to something that can be easily applied to other domains.

So now I'm back at square one - where I think a parent-child class approach is actually more appropriate. The hierarchy is shallow. Workflows are a defined set of steps and I can't imagine a case that would lead a developer to go beyond one level of inheritance.

Thoughts?
