---
layout: default
title: TIL - GIN Indexes
date: 2018-11-13 09:23 -0800
categories: [til, technical]
tags: [til, postgresql, rails]
---

Been meaning to post about this for a while, but I recently learned about Generalized Inverted Indexes. According to the [Postgres docs](https://www.postgresql.org/docs/9.6/indexes-types.html):

> GIN indexes are “inverted indexes” which are appropriate for data values that contain multiple component values, such as arrays. An inverted index contains a separate entry for each component value, and can efficiently handle queries that test for the presence of specific component values.

I plan to write a longer post about my exploration into this, including the pros and cons of denormalization, but I'll give a short overview of a problem I was trying to solve and how the GIN index came into play.

At Reflektive, I've been working on a user profile feature that gives each person a personalized newsfeed of the feedbacks they have sent or received.

Over time, the types of feedback we have in Reflektive has grown, and with each type of feedback, specific rules around who can see the feedback. For example, a user can give somebody private feedback, and only those two individuals should be allowed to see it. Or, Person A can write a private note about Person B, and not share it with Person B. While this is still stored as a Feedback in our database, and Person B is the recipient, Person B should not be allowed to see that Feedback.

In total, we have 9 different types of feedbacks, and unless the feedback is public, the feedback is private and shared with, at most, 3 people.

<!-- more -->

Querying for these feedbacks got quite complicated, because at the time, we were using a combination of attributes and checking associations to determine the type of feedback. This led to a class that looked something like this (actual queries not shown):

```ruby
class FeedbackReceivedByUser
  def execute
    gather_queries.inject(:union)
  end

  private

  def gather_queries
    case viewer
    when :admin admin_queries
    when :manager, :peer then manager_and_peer_queries
    when :self then profile_owner_queries
    end
  end

  def admin_queries
    [
      public_received,
      private_received,
      sent_by_viewer,
      feedback_requested_by_viewer
    ]
  end

  def manager_and_peer_queries
    [
      public_received,
      sent_by_viewer,
      feedback_requested_by_viewer
    ]
  end

  def profile_owner_queries
    [public_received, owner_sent_to_self, shared_with_owner]
  end
end
```

The `#execute` method makes use of the [ActiveRecord Union gem](https://github.com/brianhempel/active_record_union) which allows me to `UNION` together the separate queries into one.

In thinking about the main goal, I realized the question I was answering was just, which feedbacks are visible to the person looking at the profile?

This got me thinking - maybe I could use a bit of denormalization and store an array, called `visible_to` of the user IDs of the people allowed to view the private feedbacks on each feedback record (empty arrays would signify that the feedback is visible to everyone). Then, instead of writing several separate queries, I could simplify it to one query that simply said 'Fetch all feedbacks Person B has sent or received that are visible to Person A'.

I knew Postgres allowed storing array datatypes, and this seemed like a promising use case. However, I wanted to be sure that the query would be performant. I noticed another model in our code did something similar using the `ANY` operator. I could use that as well and construct the query like this:

```sql
SELECT feedbacks.* FROM feedbacks
WHERE (?) = ANY(feedbacks.visible_to)
OR feedbacks.visible_to IS NULL
```

However, in researching this, I realized the performance would be really slow on large tables as it doesn't use any index, so each record would have to be checked individually whether an ID existed in the array. I found out it was possible to use a GIN index to attain faster lookup performance. Even better, this was an available ActiveRecord option for creating a database index.

In the end, I was able to create a scope on the Feedback class called `visible_to` and use some custom SQL to make use of the GIN-indexed `visible_to` array:

```ruby
class Feedback < ApplicationRecord

  scope :visible_to, ->(user_id) do
    where('visible_to @> ARRAY[?] OR visible_to IS NULL', user_id))
  end
end
```

And my many separate queries could be simplified to:

```ruby
def viewable_received_feedbacks
  user.received_feedbacks.visible_to(query_user.id)
end
```

My new class looked something like this:

```ruby
class FeedbackReceivedByUser
  def execute
    gather_queries.inject(:union)
  end

  private

  def gather_queries
    case viewer
    when :admin admin_queries
    when :manager, :peer, :self then viewable_received
    end
  end

  def admin_queries
    [viewable_received, viewable_to_admin]
  end

  def viewable_received_feedback_query
    [viewable_received]
  end

  def viewable_to_admin
    target_user.received_feedbacks.where.not(sender: query_user)
  end

  def viewable_received
    target_user.received_feedbacks.visible_to(query_user.id)
  end
end
```

For managers, peers, and folks looking at their own profile, I have simplified the many queries down to one - it simply fetches the feedbacks that are visible to that person.

### Performance

In addition to simpler queries, I found I got a performance boost in all cases using the denormalized `visible_to` array with a GIN index. Depending on the scenario of the type of viewer, performance gains ranged between 2 to 5 times faster than `UNION`ing a bunch of separate queries.

```shell
looking at self

Comparison:
Using GIN array:         3584.5 i/s
Using old queries:       717.5 i/s - 5.00x  slower

looking at direct report

Comparison:
Using GIN array:         1063.4 i/s
Using old queries:       527.7 i/s - 2.02x  slower

looking at peer

Comparison:
Using GIN array:        1160.0 i/s
Using old queries:      566.0 i/s - 2.05x  slower
```
