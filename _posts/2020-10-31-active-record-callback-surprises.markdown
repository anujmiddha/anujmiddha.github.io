---
layout: post
title:  "Active Record Callback Surprises"
date:   2020-10-31 10:00:00 +0530
categories: ruby rails
---

![image](/assets/images/active-record/ruby_on_rails.jpeg)

Active Record is one of, if not the most, awesome tools in Rails. Its super intuitive and super powerful at the same time. Its _callbacks_ and _validations_ read naturally and provide immense functionality.

However Active Record Callbacks can have some surprising behavior, and should be used with proper thought and care. For our demonstrations, let’s consider a Post model with the following schema

```
create_table :posts do |t|
  t.string  :title
  t.string  :description
  t.boolean :published, default: false, null: false
  t.boolean :posted_on_social_media, default: false, null: false

  t.timestamps
end
```
___

### after_create_commit

As soon as a post is created, we want to publish it and post it to social media. A first thought might be to add two callbacks in the after_create_commit callback,

```
# Posts
class Post < ApplicationRecord
  after_create_commit :publish, :post_on_social_media
  
  private
  
  def publish
    puts 'publishing'
    update(published: true)
  end
  
  def post_on_social_media
    puts 'posting on social media'
    update(posted_on_social_media: true)
  end
end
```

While this might seem to be correct at first glance, this will in fact only post to social media and not publish your post at all! Here’s why,

The callbacks are called in the reverse order of their definitions. So `post_on_social_media` will be called before `publish`. While inside this method, the code pushes an update to the record, thereby halting the `create` callback chain and triggering any `update` callbacks.

Lets define the specs for our expectations,

```
require 'rails_helper'
RSpec.describe Post, type: :model do
  describe 'Callbacks' do
    subject(:post) { Post.create(title: 'Awesome title', description: 'Awesome description') }

    it { is_expected.to have_attributes(posted_on_social_media: true) }
    it { is_expected.to have_attributes(published: true) }
  end
end
```

While the first test passes, the second one fails.

```
$ bundle exec rspec
...
Failed examples:
rspec ./spec/models/post_spec.rb:8 # Post Callbacks is expected to have attributes {:published => true}
```

If we read the official [guide][1] for Rails, this was to be expected. It clearly mentions to avoid calling `update` in the `after_commit` callbacks.

> Avoid updating or saving attributes in callbacks. For example, don’t call update(attribute: "value") within a callback. This can alter the state of the model and may result in unexpected side effects during commit. Instead, you can safely assign values directly (for example, self.attribute = "value") in before_create / before_update or earlier callbacks.

____
  
So how should we define the callbacks in this case?

The first thing to notice is the choice of the event for the callback, `after_create_commit`. Active Record callbacks (except `after_commit`) are executed in a transaction, which can be rolled back if there’s an exception. However, if our system needs to interact with the world outside our database in the callbacks, then we need to be sure that the transaction has been committed before actually interacting with the outside world, as those changes cannot be rolled back.

In our case, the method `post_on_social_media` interacts with the world outside, so `after_create_commit` is the right place for it. However `publish` just updates a column in the database, and should instead be in a `before_create` or `after_create` callback. On updating our callbacks as below,

```
class Post < ApplicationRecord
  before_create :publish
  after_create_commit :post_on_social_media
  after_update_commit :print_update_log
  
  private
  
  def publish
    puts 'publishing'
    self.published = true
  end
  
  ...
end
```

our tests are now passing.

```
$ bundle exec rspec
=> 2 examples, 0 failures
```
___
### Using both after_create_commit and after_update_commit

The official guide says

> Using both after_create_commit and after_update_commit in the same model will only allow the last callback defined to take effect, and will override all others.

While this is true if both of these callbacks refer to the same method,

```
class Post < ApplicationRecord
  after_create_commit :print_update_log
  after_update_commit :print_update_log
  
  private
  
  def print_update_log
    puts 'post was updated'
  end
end

# Prints nothing
@post = Post.create

# Updating post
@post.save
=> post was updated
```

but if we changed this to call different methods in both callbacks,

```
class Post < ApplicationRecord
  after_create_commit :print_create_log
  after_update_commit :print_update_log
  
  private
  
  def print_create_log
    puts 'post was created'
  end
  
  def print_update_log
    puts 'post was updated'
  end
end

# Create
@post = Post.create
=> post was created

# Update
@post.save
=> post was updated
```

In most cases you will have different callbacks for create and update, which might make you believe that the usage has no issues. But there’s this edge case that you should be careful of, lest it catch you unawares.

To close, with proper thought and care, Active Record can be used safely and super charge your applications.

[1]: https://edgeguides.rubyonrails.org/active_record_callbacks.html#destroying-an-object