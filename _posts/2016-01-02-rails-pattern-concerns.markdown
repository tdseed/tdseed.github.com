---
layout: post
title:  "Rails 4 Patterns - Concerns"
date:   2016-01-02 23:29:25
categories: technology
author: tank
---

本文为Rails 4 Patterns笔记摘要系列3-Concerns

Concerns

* 3.1 Concerns
* 3.2 Model Concerns I
* 3.3 Model Concerns 2
* 3.4 Controller Concerns I
* 3.5 Controller Concerns II

> Both the User and Item models have many reviews, and they also have the exact same reviews_rating method.
Let's move the association code and the reviews_rating method code to the Reviewable concern.
Make sure to use ActiveSupport::Concern.

```ruby
module Reviewable

end

  # user.rb
class User < ActiveRecord::Base
  has_many :reviews, as: :reviewable, dependent: :destroy

  def reviews_rating
    (reviews.positive.count / reviews.approved.count.to_f).round(2)
  end
end

  # item.rb
class Item < ActiveRecord::Base
  has_many :reviews, as: :reviewable, dependent: :destroy

  def reviews_rating
    (reviews.positive.count / reviews.approved.count.to_f).round(2)
  end
end

  # review.rb
class Review < ActiveRecord::Base
  belongs_to :reviewable, polymorphic: true

  MINIMUM_POSITIVE_RATING = 5
  scope :positive, -> { where('is_approved = ? AND rating > ?', true, MINIMUM_POSITIVE_RATING) }
  scope :approved, -> { where(is_approved: true) }
end


  # 重构为
module Reviewable
  extend ActiveSupport::Concern

  included do
    has_many :reviews, as: :reviewable, dependent: :destroy
  end

  def reviews_rating
    (reviews.positive.count / reviews.approved.count.to_f).round(2)
  end

end
```

> We now have a new class method called with_no_reviews, and it looks pretty much the same for both models.
Extract this method to the Reviewable concern, and make sure it's included inside of the inner module ClassMethods so that it's automatically detected by ActiveSupport::Concern.

```ruby
module Reviewable
  extend ActiveSupport::Concern

  included do
    has_many :reviews, as: :reviewable, dependent: :destroy
  end

  def reviews_rating
    (reviews.positive.count / reviews.approved.count.to_f).round(2)
  end

  module ClassMethods
  end
end

  # item.rb
class Item < ActiveRecord::Base
  include Reviewable

  def self.with_no_reviews
    where('id NOT IN (SELECT DISTINCT(reviewable_id) FROM reviews WHERE reviewable_type = ?)', self.name)
  end
end

  # user.rb
class User < ActiveRecord::Base
  include Reviewable

  def self.with_no_reviews
    where('id NOT IN (SELECT DISTINCT(reviewable_id) FROM reviews WHERE reviewable_type = ?)', self.name)
  end
end


  # 重构为
module Reviewable
  extend ActiveSupport::Concern

  included do
    has_many :reviews, as: :reviewable, dependent: :destroy
  end

  def reviews_rating
    (reviews.positive.count / reviews.approved.count.to_f).round(2)
  end

  module ClassMethods
    def with_no_reviews
      where('id NOT IN (SELECT DISTINCT(reviewable_id) FROM reviews WHERE reviewable_type = ?)', self.name)
    end
  end
end
```


> We have two controllers which allow users to download a file from your server.
Move the send_file method to a send_image method within the ImageExportable module, which accepts an image_path variable.
Don't forget to replace the item/user.image with an image_path variable.

```ruby
module ImageExportable

end

  # users_controller.rb
class UsersController < ApplicationController

  def file
    user = User.find(params[:id])
    send_file(user.image, type: 'image/jpeg',  disposition: 'inline')
  end
end

  # items_controller.rb
class ItemsController < ApplicationController

  def file
    item = Item.find(params[:id])
    send_file(item.image, type: 'image/jpeg',  disposition: 'inline')
  end
end

  # 重构为
module ImageExportable
  def send_image(image_path)
    send_file(image_path, type: 'image/jpeg',  disposition: 'inline')
  end
end
```

> Now that you have your ImageExportable module, go ahead and include it in the UsersController.
Replace the send_file method call with the send_image method from the ImageExportable module you created in the previous challenge.

```ruby
class UsersController < ApplicationController

  def file
    user = User.find(params[:id])
    send_file(user.image, type: 'image/jpeg',  disposition: 'inline')
  end
end

  # 重构为
class UsersController < ApplicationController
  include ImageExportable

  def file
    user = User.find(params[:id])
    send_image(user.image)
  end
end
```
