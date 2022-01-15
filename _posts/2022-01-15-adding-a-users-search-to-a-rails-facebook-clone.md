---
layout: post
title:  Adding a Users Search to a Rails Facebook Clone.
date:   2022-01-15 00:06:25 -0500
image:  '/images/search.jpg'
tags:   [rails, omniauth, authentication, ngrok]
read_time: 5
comments: true
---

## Introduction

I recently completed the core functionality of my Rails Facebook clone project, [Gembook](https://github.com/joe-mccann-dev/gembook). Today, I decided to add a rudimentary search form to the `Users#index` page, allowing users to search for other users by name. I knew this would involve playing around with `routes.rb`, and that I would need a form to submit a `get` request to the route created in `routes.rb`.

### What does this post cover?

We will cover the steps necessary to implement a basic search feature. This will make it easier for users to find their friends.

This post will cover the following:

1. Adding a route to handle search requests
2. Creating the `User.search` method.
3. Handling the search request in the `UsersController`
4. Using the Rails' `form_with` helper and displaying search results.
5. Testing this behavior with RSpec.

## Step 1 | Adding a Route to Handle Search Requests

I am using resourceful routes for this application. Since we are searching for users it makes sense to nest the `search` route within `resources :users`. For an excellent guide to routing in Rails, see the [official guide](https://guides.rubyonrails.org/routing.html).

Let's break down `get 'search', to: 'users#index', on: :collection`. The snippet below allows get requests to `/users/search` and dispatches those requests to the index action of the `UsersController`. It will also create the `search_users_url` and `search_users_path` route helpers, which we will use later.

{% highlight ruby %}
  # config/routes.rb
  resources :users, only: [:index, :show] do
    get 'search', to: 'users#index', on: :collection
    resource :profile
  end
{% endhighlight %}

Before submitting search requests, we need a way for requests to be processed. We are searching for users, so it makes sense that we involve the User model, as that is the model that knows everything about a given user, including their name, which is how a user will search for other users.

## Step 2 | Creating the User.search method

My User model includes a `first_name` and `last_name` column

The class method 

{% highlight ruby %}
  # app/models/user.rb
  def self.search(query)
    return unless query

    name = query.strip.downcase.split
    where('lower(first_name) = ? OR lower(last_name) = ?', name.first, name.last)
  end
{% endhighlight %}

Photo by <a href="https://unsplash.com/@markuswinkler?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText">Markus Winkler</a> on <a href="https://unsplash.com/s/photos/search?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText">Unsplash</a>
  