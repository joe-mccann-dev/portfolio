---
layout: post
title:  Adding a Users Search to a Rails Facebook Clone.
date:   2022-01-15 00:06:25 -0500
image:  '/images/search.jpg'
tags:   [rails, rspec, databases, routing, forms]
read_time: 5
comments: true
---

Photo by <a href="https://unsplash.com/@markuswinkler?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText">Markus Winkler</a> on <a href="https://unsplash.com/s/photos/search?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText">Unsplash</a>

## Introduction

I recently completed the core functionality of my Rails Facebook clone project, [Gembook](https://github.com/joe-mccann-dev/gembook). I recently decided to add a rudimentary search form to the `Users#index` page, allowing users to search for other users by name. I knew this would involve playing around with `routes.rb`, and that I would need a form to submit a get request to the route created in `routes.rb`.

### What does this post cover?

We will cover the steps necessary to implement a basic search feature. This will make it easier for users to find their friends, but the approach used here can be applied to any application where a user might want to search a collection.

This post will cover the following:

1. Adding a route to handle search requests
2. Creating the `User.search` method
3. Handling the search request in the `UsersController`
4. Submitting the  Search Form and Displaying Results
5. Testing this behavior with RSpec

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

## Step 2 | Creating the Search method

My User model includes both a `first_name` and `last_name` column. The `User.search` method will perform an Active Record query using the `query` argument. Unnecessary database calls are prevented by the guard clause `return unless query`. The `query` argument is assigned to `name`, stripped away of any whitespace, converted to lower-case, and split into an array. Users can then search by first or last name and the method will return matches for either or both. The `lower` function is an [SQL function](https://www.w3schools.com/sql/func_sqlserver_lower.asp) that converts a string to lower-case.

{% highlight ruby %}
  # app/models/user.rb
  def self.search(query)
    return unless query

    name = query.strip.downcase.split
    where('lower(first_name) = ? OR lower(last_name) = ?', name.first, name.last)
  end
{% endhighlight %}

## Step 3 | Handling the search request in the UsersController

Because Rails' controllers handle all incoming requests from the router, when a get request is submitted to `/users/search`, the request will be dispatched to the `users#index` action as shown above. The controller also can create an instance variable, in this case `@results`, that is accessible to the View.

Below, the controller sends a message to the `User` class (`:search`) with an argument from the params hash, `params[:query]` and assigns the results to the `@results` instance variable.

{% highlight ruby %}
  # app/controllers/users_controller.rb
  def index
    # other instance variables etc...
    @results = User.search(params[:query])
  end
{% endhighlight %}

## Step 4 | Submitting the Search Form and Displaying Results

### Submitting the Search Form

Here is where all the above setup pays off. Using the `form_with` helper, we can submit a get request to the `search_users_path`, triggering the index action of the `UsersController` (remember our route: `get 'search', to: 'users#index', on: :collection`). As seen in Step 3, the index action includes a call to the `User.search` method.

{% highlight erb %}
  <%= form_with(url: search_users_path, method: :get) do |f| %>
    <%= f.text_field :query, placeholder: 'Search for users by name.', required: true %>
    <%= f.submit 'Search' %>
  <% end %>
{% endhighlight %}

Upon submission of the form, the `UsersController` will have access to the params hash, specifically `params[:query]`. For example, consider a user named John Hancock. His friend, Thomas Jefferson, wants to search for him using Gembook. Mr. Jefferson enters 'John' into the text field. The query of 'John' gets passed to the `UsersController` as `{"query"=>"John"}` and an Active Record query is performed in the User model.

### Displaying Results

To display the search results, we can create a partial: `app/views/users/_result.html.erb`. This allows utilization of the `:collection` option. For more on this, see the [official guide](https://guides.rubyonrails.org/layouts_and_rendering.html#rendering-collections).

{% highlight erb %}
<!-- app/views/users/index.html.erb -->
<% if params[:query].present? %>
  <h2>Search Results</h2>
  <ul>
    <%= render partial: 'result', collection: @results %>
  </ul>
  <%= content_tag(:em, "No users found") if @results.none? %>
<% end %>
{% endhighlight %}

What you display in your result partial will depend on your application.

Mine looks like this:

{% highlight erb %}
 <!-- app/views/users/_result.html.erb -->
  <% if current_user.friends.include?(result) %>
    <%= render partial: 'friend', locals: { friend: result } %>
  <% else %>
    <%= render partial: 'user', locals: { user: result } %>
  <% end %>
{% endhighlight %}

The result object is sent to a different partial depending on whether or not the current user is friends with the search result. If they are friends, an "unfriend" button will appear. If they are not friends, an "Add Friend" button will appear.


## Step 5 | Testing with RSpec

### Unit testing User.search

Before writing a system spec, let's make sure this method behaves as expected with a unit test in `user_spec.rb`. A variable called `expected results` holds an Active Record relation returned from the test database. A variable called `search_results` holds the ActiveRecord relation returned by the `User.search` method. We then assert that `user`, who is a member of `expected_results`, is included in `search_results`.

{% highlight ruby %}
 # spec/models/user_spec.rb
  RSpec.describe User, type: :model do
    before do
      Rails.application.load_seed
    end

    let!(:user) { User.first }

    # several other specs etc...

    describe '.search' do
      it 'accepts a query string and returns user results' do
        expected_results = User.where(first_name: user.first_name, last_name: user.last_name)
        search_results = User.search(user.full_name)
        user = expected_results.first
        expect(search_results).to include(user)
      end
    end
  end
{% endhighlight %}

### Writing a System Spec

For the system spec, we want to simulate user interaction with the application. To achieve this, we can create two test users, Thomas and John, and have Thomas search for John. We will then assert our expectation that a valid search returns a matching user and that a search for a non-existent user comes up empty.

{% highlight ruby %}
# spec/system/search_users_spec.rb

require 'rails_helper'

RSpec.describe "SearchUsers", type: :system do
  before do
    driven_by(:rack_test)
  end

  let!(:user) { User.create(first_name: 'Thomas', last_name: 'Jefferson', email: 'thomas@jefferson.com', password: 'foobar') }
  let!(:other_user) { User.create(first_name: 'John', last_name: 'Hancock', email: 'john@hancock.com', password: 'foobar') }

  describe 'searching for a user' do
    context 'a user is logged in at users#index' do
      before do
        login_as(user, scope: :user)
        visit users_path
      end

      it 'allows them to enter a query and shows them results' do
        query = other_user.first_name
        fill_in 'query', with: query
        click_on 'Search'
        expect(page).to have_content('Search Results')
        expect(page).to have_content(other_user.full_name)
      end

      it "Shows 'No users found' if there are no matches" do
        query = 'Joe'
        fill_in 'query', with: query
        click_on 'Search'
        expect(page).to have_content('Search Results')
        expect(page).to have_content('No users found')
      end
    end
  end
end

{% endhighlight %}

## Conclusion

In this post, I attempted to show the process of adding a user search feature. In retrospect, I think a feature like this would be an excellent candidate for TDD. If I were to use TDD to implement this feature, I would probably start with the unit test and then write the method until the test passed, refactoring where appropriate. Next, I would write the system specs, and finally, write the form and view code to make them pass.

Writing your own search method is a good way to grapple with core Rails functionality. This feature utilizes the  Rails router, Model, View, and Controller, all of which are working hard to display search results to your users.

I hope you enjoyed this post. If there is anything I did incorrectly or that could be improved, please let me know in the comments! I love to see different and better approaches to the same problem. Thanks and happy coding!