---
layout: post
title:  Removing Logic from a Rails View
date:   2021-12-22 19:20:16 -0500
image:  '/images/unsplash-rails-aerial.jpg'
tags:   [rails, views]
---

## Introduction

While some logic in Rails views is inevitable, recently I have been trying to move any unnecessary logic in the view to the model or controller--especially if it's a database query. This problem came into *view* while I was coding a Friendship "Accept" button in a Rails Facebook clone.
I had it setup so that users could accept or decline friend requests from the Notifications#index page. The problem was I had to locate the respective friendship object from within Notifications#index. This required a database query. Once I wrote the correct database query in the view to find the Friendship object associated with a notification, I started thinking of ways to remove it from the view. Below is how I approached the problem.

Every notification references a `sender` and a `receiver`, both belonging to the User class. Similarly, a Friendship object is initiated with a `sender` and a `receiver`. If a user has many friend requests on their notifications page, it is necessary to determine *which specific* friend request is being accepted or declined when the receiver clicks "Accept" or "Decline", respectively.

Therefore, when rendering the notifications collection, we can find the soon-to-be-accepted or soon-to-be-declined Friendship with `notification.sender` and
`notification.receiver`. The "Accept" button is as follows:

### Initial "Accept" Button

{% highlight erb %}
  <% friendship = Friendship.find_by(sender_id: notification.sender.id,
                                     receiver_id: notification.receiver.id) %>
  <%= button_to "Accept",
              friendship_path(friendship),
              method: :put,
              params: {  notification: { time_sent: notification.time_sent },
                         friendship: { status: 'accepted', sender_id: notification.sender.id } }
                         %>
{% endhighlight %}

We can eliminate the need for the local variable by creating a hash of all friend requests sent to the current user, mapped to the `sender_id`. The current user will always be the receiver in this situation and to find the Friendship request sent to the current user, we only need the `sender_id`. I thought it made sense to make this method an instance method on User but there are probably better ways to do this. Let me know in the comments! Here are the relevant models and associations.

## Relevant Models

### User, Friendship, Notification

{% highlight ruby %}
  class User < ApplicationRecord
    has_many :sent_notifications,
            class_name: 'Notification',
            foreign_key: 'sender_id',
            dependent: :destroy
    has_many :received_notifications,
            class_name: 'Notification',
            foreign_key: 'receiver_id',
            dependent: :destroy

    has_many :sent_pending_requests, -> { friendship_pending },
            class_name: 'Friendship',
            foreign_key: 'sender_id',
            dependent: :destroy
    has_many :received_pending_requests, -> { friendship_pending },
            class_name: 'Friendship',
            foreign_key: 'receiver_id',
            dependent: :destroy
    # ...
  end
{% endhighlight %}

{% highlight ruby %}
class Notification < ApplicationRecord
  belongs_to :sender, class_name: 'User'
  belongs_to :receiver, class_name: 'User'
  #...
end
{% endhighlight %}

{% highlight ruby %}
class Friendship < ApplicationRecord
  enum status: %i[pending accepted declined]

  belongs_to :sender, class_name: 'User'
  belongs_to :receiver, class_name: 'User'

  scope :friendship_pending, -> { where(status: :pending) }
  #...
end
{% endhighlight %}

And now the method that maps a sender id to a Friendship object:

{% highlight ruby %}

  class User
  #...associations, etc.
    def requests_via_sender_id
      requests = received_pending_requests
      sender_ids = requests.pluck(:sender_id)
      sender_ids.to_h do |id|
        [id, requests.find_by(sender_id: id)]
      end
    end
  end
{% endhighlight %}

Let's say `received_pending_requests.count == 1` at the moment. A user with an id of 3 has sent `current_user` a request.

{% highlight ruby %}

current_user.requests_via_sender_id =>
{3=>
  #<Friendship:0x000055f71022ec20
   id: 16,
   status: "pending",
   sender_id: 3,
   receiver_id: 1 }
{% endhighlight %}

The good thing about doing it this way is that the controller can now request this information from the User model prior to rendering the Notifications#index view:

{% highlight ruby %}
  class NotificationsController < ApplicationController
    def index
      @notifications = current_user.received_notifications.includes(%i[sender receiver])
      @friendships = current_user.requests_via_sender_id # => returns a hash
    end
  end
{% endhighlight %}

Finally, we can return to our "Accept" button which has access to the current notification being rendered.
Note the absence of the `friendship` local variable:

### Final "Accept" Button

{% highlight erb %}
  <%= button_to "Accept",
              # accessing @friendships as declared in NotificationsController
              friendship_path(@friendships[notification.sender.id]),
              method: :put,
              params: {  notification: { time_sent: notification.time_sent },
                         friendship: { status: 'accepted', sender_id: notification.sender.id } }
                         %>
{% endhighlight %}

## Conclusion

Another added benefit of this approach is improved readability. Which friendship are we updating? The one the sender of a notification requested -->
`friendship_path(@friendships[notification.sender.id])`. This is arguably more expressive than `friendship_path(friendship)`, because it provides the context in which we are searching for a Friendship object.

Balancing responsibilities between the Model, View, and Controller is sometimes challenging, but making your way towards a solution piece by piece can be very educational and rewarding.

This was my first blog post. Thank you so much for reading. Having finished it I realize the benefit it has in regards to understanding. I hope to write more posts in the near future.

Photo by <a href="https://unsplash.com/@hookie1001?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText">Lawrence Hookham</a> on
<a href="https://unsplash.com/s/photos/rails-view?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText">Unsplash</a>
  