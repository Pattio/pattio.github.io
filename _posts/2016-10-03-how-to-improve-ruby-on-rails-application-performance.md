---
layout: post
title: How to Improve Your Ruby on Rails Application Performance
tags: [ruby, rails, performance]
categories:
- blog
---

Ruby on Rails is a great framework, because it lets you start quickly and move fast. However, in recent years Rails attracted a lot of negativity, because of its performance. In this post, I will try to go through few techniques to show you how to improve your Ruby on Rails application performance.

All tests were made on my local machine in development mode. For more accurate results, testing should be done in production mode.

# Active Record

`Active Record` is known to be one of the biggest bottleneck of Ruby on Rails. I will show you 2 cases where `Active Record` decreases overall application performance and how to fix them.

## N+1 queries problem

This problem occurs when you are fetching `Active Record` model which has association with other Active Record model. Imagine you have blog engine written in Ruby on Rails. You have `Post` model that has `belongs_to` relationship to `User` model. `User` model which has two relationships: `has_many` relationship to `Post` model and `has_one` relationship to `Role` model. And of course `Role` model has `belongs_to` relationship to `User` model. In other words, User can have many posts and only one role and each post has one user (author). Now assume you have to show 10 posts on blog index page. It might look easy just fetch 10 records and then display all information you need:

{% highlight ruby %}
class BlogController < ApplicationController
  def index
    # Fetch posts
    @posts = Post.limit(10)
  end
end
{% endhighlight %}

{% highlight ruby %}
<% @posts.each do |post| %>
    <%= post.title %>
    <%= post.created_at %>
    <%= image_tag(post.thumbnail, class: "thumbnail") %>
    <%= post.body %>
    <%= post.user.name %>
    <%= post.user.role.name %>
<% end %>
{% endhighlight %}

![21 query]({{ "/assets/images/21-query.png" | absolute_url }}){:class="scale-with-grid"}

However, if you look at console log you can see that first we fetch all posts from database and then we are making two new queries for each post. 10 * 2 + 1 = **21 queries** in total. It is because `post.user.name` makes new query to get user name and `post.user.role.name` makes another query to get user role. We can achieve better results by preloading data that we are going to use later. In this case we include users in our `Post` query:


{% highlight ruby %}
class BlogController < ApplicationController
  def index
    # Preload users
    @posts = Post.includes(:user).limit(10)
  end
end
{% endhighlight %}

![4 queries]({{ "/assets/images/4-queries.png" | absolute_url }}){:class="scale-with-grid"}

When we open console log again we can see only **4 queries**! As a result, page loads much faster. But we can do even better. When you look at console log you can see that we are still making 2 queries to load our roles. If we would have even more roles, not just administrator and user we would have to make even more queries. Let’s fix this problem by preloading roles:

{% highlight ruby %}
class BlogController < ApplicationController
  def index
    @posts = Post.includes(user: :role).limit(10)
  end
end
{% endhighlight %}

![3 queries]({{ "/assets/images/3-queries.png" | absolute_url }}){:class="scale-with-grid"}

Now we only used **3 queries**, and the question in your mind probably is, can we do even better? And the answer is of course. As you can see we are selecting all columns from user table, however we only need user name, we don’t need user password, bio and avatar. Same rule applies to role table, we don’t need role description and moderate value (boolean value which show if user can edit posts). So, using `join`, we can put all records in one table and only take columns we actually need (user name and role name):

{% highlight ruby %}
class BlogController < ApplicationController
  def index
    @posts = Post.joins(user: :role).select("posts.*, users.name as user_name, roles.name as role_name").limit(10)
  end
end
{% endhighlight %}

![1 query]({{ "/assets/images/1-query.png" | absolute_url }}){:class="scale-with-grid"}

In our `index.html.erb` file we have to change `post.user.name` to `post.user_name` and `post.role.name` to `post.role_name`. Now we only made **1 query** and we got only those values that we need. 

### Final results

Queries:
* From **21** queries to just **1** query

Time spent in `ActiveRecord`:
* From **7.2ms** to just **1.1ms**


So, even though `Active Record` makes our life easier, sometimes to achieve best results we have to use some of the SQL as well. However, **we should not rewrite every query in SQL**. We only optimize those queries, that take a long time to finish and get called a lot.

## Fetching all records at once

Sometimes we need to get all the records and do something with them. For example, your server cron calls `cronjob_update_controller.rb` every 24 hours. This controller fetches all posts and then checks if post has any comments, if yes controller awards points to post author for each comment on the post. One of the simplest ways to achieve that would be, fetching all records and then looping trough them:

{% highlight ruby %}
@posts = Post.all
@posts.each do |post|
  if post.comments.count > 0
    post.author.points = points_function(post.comments.count)
  end
end
{% endhighlight %}

![Fetching all records at once]({{ "/assets/images/fetching-all-records-at-once.png" | absolute_url }}){:class="scale-with-grid"}

It works perfectly fine, if you have not that many posts. But what happens when you have several thousands of posts? You will try to instantiate all the objects at once, which means you will have to allocate memory for all of them at the same time. This will be very slow, but the biggest concern is that it might even crash your application, because of RAM shortage. To avoid huge memory usage, you should fetch records in batches. `Active Record` has method `find_each` which by default fetches records with batches of size 1000. It retrieves 1000 records, you do something with those records, then it retrieves another 1000 records and process repeats till you get all the records from your database. This way you don’t need to allocate memory for all records at once. And if you won’t keep reference to the post, each time loop iterates, ruby garbage collector will remove old posts from memory and will keep application memory usage low.

![Fetching records in batches]({{ "/assets/images/fetching-records-in-batches.png" | absolute_url }}){:class="scale-with-grid"}

The important result here is memory usage of Ruby on Rails app:

**Idle:**
![Idle]({{ "/assets/images/idle.png" | absolute_url }}){:class="scale-with-grid"}

**Fetching all records at once:**
![Fetching all records at once, ram usage]({{ "/assets/images/fetching-all-records-at-once-ram.png" | absolute_url }}){:class="scale-with-grid"}

**Fetching records in batches:**
![Fetching records in batches, ram usage]({{ "/assets/images/fetching-records-in-batches-ram.png" | absolute_url }}){:class="scale-with-grid"}

I was using **11028** post records as my test data, with bigger data sets difference would be bigger. But even **32 MB** we saved here, counts when your server is under big constraints.

# Cache

Cache frequently accessed content. You don’t need to cache every single bit of your application, but you should cache parts that get most requests and do some heavy lifting. Good news is that fragment caching is included in Ruby on Rails framework. You just need to choose cache backend. 

First option is `ActiveSupport::FileStore` and second option is `ActiveSupport:MemoryStore`. The biggest difference between them is that first option stores caches on the filesystem and second one keeps cache on RAM.


FileStore advantages:
* All processes on same host can access cache.
* Disk space cost less than RAM.

MemoryStore advantages: 
* Accessing RAM is much faster than accessing disk.

So, if your cache is not going to be big you should probably use MemoryStore. 

### Why you should use cache?

Well for starters, by using cache you could have solved **n + 1 queries** problem. Just by adding `cache` method to post variable, we have cache working:   

{% highlight ruby %}
<% @posts.each do |post| %>
  <% cache(post) do %>
    <div class="post">
        # Post information
    </div>
  <% end %>
<% end %>
{% endhighlight %}


It's that simple. Now if posts have not changed, application just reads fragment for each post and displays posts from those fragments. If post have changed, we just update our fragment for those posts that have changed. 

To make our cache even more advanced, we can add cache method to all posts as a group:
{% highlight ruby %}
<% cache(["posts", @posts.map(&:id), @posts.maximum(:updated_at)]) do %>
  <% @posts.each do |post| %>
    <% cache(post) do %>
    <div class="post">
      # Post information
    </div>
    <% end %>
  <% end %>
<% end %>
{% endhighlight %}

Now we are giving cache method string "posts", each post that we are going to display ID and value when post was updated last time. From all of those values cache method will create name to represent cache fragment. Now we don’t need to read 50 different cache fragments, for each post. Instead we only need to read 1 fragment. This technique we are using is called **Russian doll** (as there are layers of cache).

How this solves **n + 1 queries** problem? Well, now on first page load we put data to cache and after that, every time we load page, we only make one query to database to get posts IDs and maximum updated at value. 

But the real beauty and importance of cache comes when we are doing some hard processing. Imagine that for each post we load we do some data processing takes some time. To make things easier inside each post I will use `sleep` method with value of 0.1sec (to represent something that takes 0.1sec to finish for each post). Now I will open index page:

![Heavy task no cache]({{ "/assets/images/heavy-task-no-cache.png" | absolute_url }}){:class="scale-with-grid"}

As you can it takes more than **5sec** to load page. The reason why it takes so long is, because we have **50 posts** and each of them have to do something for **0.1sec** (we are using `sleep` method for that) so the total load time will always be greater than 5sec. However, when we turn caching on, look what happens:

![Heavy task with caching]({{ "/assets/images/heavy-task-with-caching.png" | absolute_url }}){:class="scale-with-grid"}

It takes only **0.034sec** to load page. It is because we load all posts from our cache and we don’t need to do heavy lifting each time post is loaded. 

However, you shouldn't cache every single bit of your application, because using cache for wrong purpose can make your application slower.

# Conclusion

So, next time when you are about to blame some framework for being slow stop and try to think for a second. Maybe application is slow not because of the framework, but because of how are you using the framework. And maybe 
> I will just buy more servers

is not something you should keep in your mind as a solution to performance problems.

And finally, even though performance optimizations are great, keep in mind that you can’t just try to optimize every single bit of your application. You should run benchmarks and see which specific parts of your application are slow. After you find those parts you should search for what is causing slowness and how to fix that. Otherwise, if you will try to optimize every single piece of your application, you will spend your time inefficiently and barely increase your application performance. 


---
### References:

* [Ruby on Rails Guides N+1 queries](http://guides.rubyonrails.org/active_record_querying.html#eager-loading-associations)
* [Ruby on Rails API Batch processing](http://api.rubyonrails.org/classes/ActiveRecord/Batches.html)
* [Nate Bekopec, The Complete Guide to Rails Caching](http://www.nateberkopec.com/2015/07/15/the-complete-guide-to-rails-caching.html)

