---
layout: default
title: "Refactoring Fat Models with DCI"
date: 2015-08-17 10:00
comments: true
categories: technical
tags: [rails, ruby, dci]
---

Rails developers are familiar with the maxim "fat models, skinny controllers". If logic starts creeping into our controllers, most developers seem content to push that logic into a model. But obese models can make for painful development. At Bloc, we've reached the point where this has become painful enough that I decided something needed to be done about it. After reading through Jim Gay's fantastic [Clean Ruby](http://clean-ruby.com/), I identified many areas in which our application could benefit from the [DCI](https://en.wikipedia.org/wiki/Data,_context_and_interaction) (data, context, interaction) design pattern.

DCI, according to that wikipedia link above, “separates the domain model (data) from use cases (context) and Roles that objects play (interaction)”. In Clean Ruby, Jim Gay illustrates this concept with transferring money between two accounts. The domain model is an account, which takes on the role of a transferrer of money from itself to another account, in the context of transferring money.

The situation I came across in the Bloc application involved populating a progress bar with data related to a student’s enrollment. This used to be fairly straightforward, but after introducing our [Track](https://www.bloc.io/web-development-bootcamp) products, in which students can take a track of two courses, it became a little more complicated.

<!-- more -->

A quick overview of the data model - students have an enrollment that has many states so that we can keep track of a student's experience over time (e.g. if a student ‘freezes’ her enrollment, an enrollment state record is created that records the student is frozen). A new enrollment state is created when a student begins the next course in her track.

We have a progress bar that shows up on a student's current roadmap. The data for the progress bar is built based on a lot of information related to a student’s enrollment. For track students, we decided to show them a progress bar for their next course, to maintain continuity and to give some helpful information, such as when they'll choose their mentor next. The complication occurs when fetching a progress instance for an enrollment state that doesn't exist yet. We needed to add in some logic that determines whether the roadmap a student is looking at is the enrollment's current roadmap or a roadmap for a future enrollment state.

Naturally, I began writing instance methods in the model for making these comparisons. Eventually, I had five new methods on the model that help determine whether to return a progress instance for the current roadmap or for a stub that represents a future enrollment state.

I could have stopped there, but I knew that these methods don't really need to be cluttering up our Enrollment model. They're only needed in the context of fetching a progress instance.

Recognizing a context, I decided to create one describing exactly that, calling the class `FetchingProgressInstance`.

```ruby
class FetchingProgressInstance
  attr_reader :enrollment, :roadmap

  def initialize(enrollment, roadmap)
    @enrollment = enrollment
    @roadmap = roadmap
    assign_progress_fetcher(@enrollment)
  end

  def fetch_progress
    enrollment.progress_for(roadmap)
  end

  private

  def assign_progress_fetcher(enrollment)
    enrollment.extend(ProgressFetcher)
  end

  module ProgressFetcher
    def progress_for(given_roadmap)
      if roadmap == given_roadmap
        # return progress
      elsif has_roadmap?(given_roadmap)
        # determine whether the roadmap is for a previous
        # enrollment state or a future enrollment state
        # and return the progress instance for it
      end
    end

    # more methods

  end
end
```
Now these methods are encapsulated in the context in which they are needed and added to the enrollment instance at runtime, instead of adding further clutter to the Enrollment model.

The class is easily testable and will be much easier to debug should a problem arise. Instead of tracking down methods inside of a disorganized model class, all the necessary methods are kept in this single context.

From this experience, I've come up with a few steps you can take to refactor your code to a cleaner, more modular state, whether it's choosing to use DCI or some other design pattern like service objects.

### 1. Group related methods in your model

This might seem obvious, but all too easily do models become a disorganized dumping ground of instance methods. If you group instance methods, adding a comment above the group explaining their relation, you'll be able to identify the contexts in which they are used. I do this constantly as I continue coming across bits of code that have a similar theme, organizing them so that it will become easier to extract things into contexts when needed.

### 2. Create a context class

I created a folder at app/contexts to hold my context classes. I give them names that are clear about the context in which they are used, like the example in this post: `FetchingProgressInstance`.

### 3. Create a private module

Keeping the necessary instance methods in a private module that extends the object at runtime ensures the code is used only when needed and can't be arbitrarily used.

This goes a bit against the Rails convention of creating concerns that can be mixed into your models. Following this convention, you might have code that looks something like this:

```ruby
class Post
  include Taggable
  include Commentable
end
```

I could have done the same thing with our Enrollment model, with an awkwardly named module like `ProgressFetchable`. However, I have a bit of an aversion to creating mixins that are mixed in only in one class. And in any case, we don't really achieve much in the way of keeping our classes small. As Jim Gay eloquently explains:

>
  Moving our behaviors into modules only serves to organize them based upon classes and namespaces but doesn’t give us better insight into the interactions of objects in the system. What a context in our code provides for us is an environment that encapsulates the interactions we expect to see.
>>Jim Gay, Clean Ruby 1.0

And then later:

>
  If we use a library like this or if we use plain old modules, our approach still would be tying our code to the notion that the classes of the system are the focus of our program rather than the objects at runtime.
>>Jim Gay, Clean Ruby 1.0

Personally, I love the idea of using DCI to keep code in context.  

### 4. Test it out

You can create a new spec file describing the expectations of this context without cluttering the existing model spec.

### There's more

I've only scratched the surface with DCI, and performance concerns have been documented in the past when it comes to extending objects at runtime (by busting the method cache). However, there are more techniques you can use, which are documented further in Clean Ruby, such as creating wrapper classes around your object and making use of delegation and forwarding, but that's another blog post. In the meantime, I'd encourage anyone to check out Clean Ruby and experiment with DCI yourself.
