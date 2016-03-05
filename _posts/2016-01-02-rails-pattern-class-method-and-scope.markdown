---
layout: post
title:  "Rails 4 Patterns - Scopes and Methods"
date:   2016-01-02 23:27:25
categories: technology
author: tank
---

本文为Rails 4 Patterns笔记摘要系列2-Scopes and Methods

LEVEL 2 SCOPES AND CLASS METHODSLEVEL 21500 Possible PointsLevel 2 Badge
Scopes and Methods

* 2.1 Scopes and Methods
* 2.2 Extract query I
* 2.3 Extract query II
* 2.4 Class Method to Scope
* 2.5 Merging Scopes I
* 2.6 Merging Scopes II


> There's too much code in the ItemsController class.
Extract the query used in the index action of the ItemsController to a class method named featured on the Item model.

```ruby
class Item < ActiveRecord::Base

end

  # items_controller.rb
class ItemsController < ApplicationController
  def index
    @items = Item.where('rating > ? AND published_on > ?', 5, 2.days.ago)
  end
end

  # 重构为
class Item < ActiveRecord::Base
  def self.featured
    where('rating > ? AND published_on > ?', 5, 2.days.ago)
  end
end
```

> Replace the query in the index action with a call to the featured class method on the Item model.

```ruby
class ItemsController < ApplicationController
   def index
     @items = Item.where('rating > ? AND published_on > ?', 5, 2.days.ago)
   end
end

  # 重构为
class ItemsController < ApplicationController
   def index
     @items = Item.featured
   end
end
```

> The previous code serves a search feature in our application.
However, when the user submits a blank keyword, this code raises an error.
Let's fix this by changing the two class methods below to scopes, since scopes always return a relation.

```ruby
class Item < ActiveRecord::Base
  def self.by_name(name)
    where(name: name) if name.present?
  end

  def self.recent
    where('created_on > ?', 2.days.ago)
  end
end

  # 重构为
class Item < ActiveRecord::Base
  scope :by_name, ->(name){ where(name: name) if name.present?}
  scope :recent, ->{ where('created_on > ?', 2.days.ago) }
end
```

> In the Item.recent scope, use the joins and merge methods to combine the Review.approved scope.

```ruby
class Item < ActiveRecord::Base
  has_many :reviews
  scope :recent, ->{
    where('published_on > ?', 2.days.ago)
      .joins(:reviews).where('reviews.approved = ?', true)
  }
end

  # review.rb
class Review < ActiveRecord::Base
  belongs_to :item
  scope :approved, -> { where(approved: true) }
end

  # 重构为
class Item < ActiveRecord::Base
  has_many :reviews
  scope :recent, ->{
    where('published_on > ?', 2.days.ago)
      .joins(:reviews).merge(Review.approved)
  }
end
```

> When upgrading to Rails 4, you run into this bug because Rails no longer automatically removes duplications from SQL conditions.
Please fix the code below.

```ruby
Review.relevant.pending_approval

  # 重构为
Review.relevant.merge(Review.pending_approval)
```
