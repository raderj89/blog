---
layout: default
title: "Practical Object-Oriented Refactoring"
date: 2015-01-15 12:10
comments: true
categories: technical
tags: [ruby, object oriented programming]
---

*This post first appeared on the [Bloc blog](http://code.bloc.io/practical-object-oriented-refactoring/) in January 2015*

As code bases grow, the need to refactor grows with it. A couple weeks ago, I saw an opportunity to refactor Bloc's scheduled emails, which we activate in a Rakefile.

The Rakefile looked something like this:

```ruby
task :send_greeting_email => :environment do
  Enrollment.starting_in_the_next_two_weeks.each do |enrollment|

    EnrollmentMailer.program_coordinator_greeting(enrollment).deliver
  end
end

task :send_get_started_emails => :environment do
  starting_enrollments = Enrollment.where('course_start_date > ? AND course_start_date < ?', Time.now, Time.now).where(sent_get_started_email: false)

  starting_enrollments.each do |enrollment|
    EnrollmentMailer.get_started(enrollment).deliver
  end
end

task :send_final_day_email => :environment do
  Enrollment.active.graduating_on(Date.today).each do |enrollment|
    EnrollmentMailer.graduation_letter(enrollment).deliver
    MentorMailer.confirm_grad(enrollment).deliver
  end
end
```

Plus several more notifications.

I wanted to test the code in this file, but as it is, it's not very friendly to testing. However, there was a pattern among all these scheduled emails - grabbing specific enrollments and performing certain email actions. Sounds like a great use case for a [service object](http://stevelorek.com/service-objects.html).

<!-- more -->

Here was my first go-round:

```ruby
class Services::MassEnrollmentMailer
  attr_reader :enrollments

  def initialize(enrollments)
    @enrollments = enrollments
  end

  def send_final_day_emails!
    enrollments.each do |enrollment|
      EnrollmentMailer.graduation_letter(enrollment).deliver
      MentorMailer.confirm_grad(enrollment).deliver
    end
  end

  def send_get_started_emails!
    enrollments.each do |enrollment|
      EnrollmentMailer.get_started(enrollment).deliver
    end
  end

  def send_greeting_emails!
    enrollments.each do |enrollment|
      EnrollmentMailer.program_coordinator_greeting(enrollment).deliver
    end
  end

  def send_two_week_notifications!
    enrollments.each do |enrollment|
      MentorMailer.two_weeks_notification(enrollment).deliver
      EnrollmentMailer.personal_grad_note(enrollment).deliver
    end
  end
```

This was a bit better. I could easily test this class. But it seemed a little repetitive to me. All I'm doing is iterating over the enrollments and then sending a bunch of emails. For some I use more than one mailer, but the overall goal is the same: take some enrollments and send some specific emails.

Also, if you've ever read [Practical Object Oriented Design in Ruby](http://www.poodr.com/), you'll notice this class doesn't manage dependencies very well. There are at least a couple different classes in the `MassEnrollmentMailer` class.

So I looked for a way to abstract all of this information out in such a way that my `MassEnrollmentMailer` was completely unaware of any other class. This is easily done with [dependency injection](http://en.wikipedia.org/wiki/Dependency_injection). My goal was to make a class that did only one thing, deliver emails, and it would use whatever mailer classes and email methods I gave to it.

Here's what this new class, which I called simply `MassMailer` because it would be able to perform mass emails on anything, looks like:

```ruby
class Services::MassMailer
  attr_reader :objects, :mailers_and_emails

  def self.deliver_mail!(objects: objects, mailers_and_emails: mailers_and_emails)
    new(objects, mailers_and_emails).send_emails!
  end

  def initialize(objects, mailers_and_emails)
    @objects = objects
    @mailers_and_emails = mailers_and_emails
  end

  def send_emails!
    mailers_and_emails.each do |mailer, email|
      objects.each do |object|
        send_email!(mailer, email, object)
      end
    end
  end

  private

    def send_email!(mailer, email, object)
      mailer.send(email, object).deliver
    end   
end
```

This is now a much simpler, more versatile, and easily testable class of code.

I initialize the class with the collection of enrollments as well as an array, which I call `mailers_and_emails`.

In the `send_emails!` method, I loop over the array and the enrollments, calling the `send_email!` method, which takes the mailer class, the email method, and the object needed to be acted upon.

Within `send_email!`, I can use the `send` method to call the email method with the object as its argument on the mailer class.

The Rakefile now looks like this for each task:

```ruby
task :send_final_day_email => :environment do
  enrollments = Enrollment.graduating_on(Date.today)

  Services::MassMailer.deliver_mail!(
    objects: enrollments,
    mailers_and_emails: [
      [EnrollmentMailer, :graduation_letter],
      [MentorMailer, :confirm_grad]
    ]
  )
end
```

You might notice that I chose to use a nested array for `mailers_and_emails`. At first, I experimented with a hash, in which the keys were the mailer classes, and their values were the methods. I then used `Object.const_get` in order to convert the key into a constant. While this is certainly a valid way of doing this, metaprogramming like this can often complicate debugging. The nested array allows me to directly pass in the class and allows the same ease of iteration.

Now the `MassEnrollmentMailer` only does *one* thing, `deliver_mail!`. It doesn't know or care about what mailer classes it needs, it just needs the classes and methods to be given to it.
