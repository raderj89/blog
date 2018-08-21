---
layout: default
title: From Service Objects To Interactors
date: 2018-08-21 11:03 -0700
categories: technical
tags: [ruby, rails]
---

*This post first appeared on the [Reflektive Engineering Blog](https://medium.com/reflektive-engineering/from-service-objects-to-interactors-db7d2bb7dfd9)*

If you've worked on any large Rails app, you've probably come across the service object pattern, a useful abstraction for handling business logic.

Recently, we've been playing with a particular implementation of this pattern using the [Interactor gem](https://github.com/collectiveidea/interactor).

To understand the benefits of interactors, it's helpful to first talk about service objects and how they're often used in Rails.

In the context of Rails, a service object is simply a Ruby object used for a specific business action. This code usually doesn't belong in the controller or model layer, because it might involve multiple data models, or it hits a third-party API.

<!-- more -->

Here's a contrived example of a service object, in the context of a user responding to a survey, where the user gets some kind of reward points for responding:

```ruby
class ReplyToSurvey
  def initialize(responder:, answers:, survey:)
    @sender = sender
    @answers = answers
    @survey = survey
  end

  def perform
    create_response
    add_reward_points
    send_notifications

    survey_response
  end

  private

  attr_reader :responder, :answers, :survey, :survey_response

  def create_response
    survey_response = responder.survey_responses.build(
      response_text: answers[:text],
      rating: answers[:rating]
      survey: survey
    )

    survey_response.save!

    @survey_response = survey_response
  end

  def add_reward_points
    sender.reward_account.balance += survey.reward_points
    sender.reward_account.save!
  end

  def send_notifications
    SurveyMailer.delay.notify_responder(responder.id)
    SurveyMailer.delay.notify_sender(survey.sender.id)

    sender.add_survey_response_notification!
  end
end
```

You would use this in the controller like this:

```ruby
class SurveyResponsesController < ApplicationController
  def create
    survey_response = ReplyToSurvey.new(responder: @responder, answers: survey_params[:answers], survey: @survey).perform

    redirect_to survey_response
  rescue => e
    flash[:alert] = "There was a problem saving your survey response"
    render :new
  end
end
```

At [Reflektive](https://www.reflektive.com/), we have made extensive use of service objects to encapsulate actions such as changing the manager of employees, launching review cycles, or processing Real-Time Feedback from Slack.

A problem we've encountered as our team and codebase has grown is a lack of established conventions for how service objects should be written, so engineers on different teams would write them in different ways. Some had many public methods, while others had only one. Some handled errors while others didn't. Some were concerned about data integrity, running anything data-related in ActiveRecord transactions in case they failed.

As a team, we discussed what we wanted from our service objects: Uniformity, data integrity, single responsibility, and error handling.

This lead us to the Interactor gem, created by the good folks over at Collective Idea. This gem provides a great API for much that we desire in service objects and more. I encourage you to read the docs as the library is quite simple and well-written.

As you can see from the example of our service object above, we don't have a very elegant way to handle errors in the above example, rescuing any errors in the controller. Also, you could argue that our `ReplyToSurveyservice` violates the single responsibility principle, since it actually does three things (creates the response, adds reward points, and sends notifications). There are plenty of ways we could solve this manually, but lets see how we could solve this using Interactors:

```ruby
class ReplyToSurvey
  include Interactor::Organizer

  organize CreateResponse, AddRewardPoints, SendNotifications
end

class CreateResponse
  include Interactor

  def call
    responder = context.responder

    survey_response = responder.survey_responses.build(
      response_text: context.answers[:text],
      rating: answers[:rating]
      survey: context.survey
    )

    if survey_response.save
      context.survey_response = survey_response
    else
      context.fail!(errors: survey_response.errors)
    end
  end
end

class AddRewardPoints
  include Interactor

  def call
    reward_account = context.sender.reward_account

    reward_account.balance += context.survey.reward_points

    unless reward_account.save
      context.fail!(errors: reward_account.errors)
    end
  end
end

class SendNotifications
  include Interactor

  def call
    SurveyMailer.delay.notify_responder(context.responder.id)
    SurveyMailer.delay.notify_sender(context.survey.sender.id)

    unless context.sender.add_survey_response_notification
      context.fail!(error: context.sender.errors)
    end
  end
end
```

*Note: The above would all be in separate files.*

And in the controller, this would look like:

```ruby
class SurveyResponsesController < ApplicationController
  def create
    result = ReplyToSurvey.call(responder: @responder, answers: survey_params[:answers], survey: @survey)

    if result.success?
      redirect_to result.survey_response
    else
      flash[:alert] = "There was a problem saving your survey response"
      render :new, locals: { errors: result.errors }
    end
  end
end
```

Using Interactors, we've split each step into its own class and we get a uniform way of handling errors at each step. Awesome!

There are a couple of highlights of this library that I want to point out:

## Organizers

Often, many things need to happen in a service. This can lead to services being called from within services - but with Organizers, you can list the sequence in which you need your interactors to be run. This encourages writing small services with single responsibilities. Also, the organize API allows for great self-documenting code:  `organize CreateOrder, ChargeCard, SendThankYou`

## Rolling Back

Some services might create data, but fail at a certain point. Interactors provide a `rollback` method that gives you a chance to undo anything that has been done. If you have a five interactors and the fourth one fails, the `rollback` method will be called on each in reverse order.

## Drawbacks of Interactor

Our team had some concerns about the Interactor gem, but we established a few conventions to address these issues:

### Lack of Class Signature

With Interactors, you don't define an `initialize` method because all parameters passed to an interactor are attached to a context object that is shared and mutated (a scary word to many programmers) between interactors. To make it easier to quickly understand what parameters we can expect the interactor to have, we recommended using a delegate call at the top of our classes:

```ruby
class CreateResponse
  include Interactor

  delegate :responder, :answers, :survey, to: :context

  def call
    survey_response = responder.survey_responses.build(
      response_text: answers[:text],
      rating: answers[:rating]
      survey: survey
    )

    if survey_response.save
      context.survey_response = survey_response
    else
      context.fail!(errors: survey_response.errors)
    end
  end
end
```

### Context Object

As I mentioned before, you can attach values to the context object. This is convenient when you have multiple interactors called one after the other using the organize method. One of the concerns we had was that it might be unclear what's in a context object if you can just attach values wherever and whenever you want.

To make this more predictable, we recommended a convention of only attaching values to the context at the end of the call method or within the `if something.save...` block, as you can see above.

Check out the gem! While its quite robust, the library is actually quite small and is extremely well documented. And let me know what strategies you prefer to encapsulate business logic.
