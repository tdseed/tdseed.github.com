---
layout: post
title:  "rails model refactor"
date:   2016-01-01 23:27:25
categories: technology
author: tank
---

LEVEL 1 MODELS
LEVEL 1

Level 1 Badge
Rails 4 Models

* 1.1 Rails 4 Models
* 1.2 Skinny Controllers I
* 1.3 Skinny Controllers II
* 1.4 ActiveRecord Callbacks
* 1.5 Non-AR Models - Part I
* 1.6 Non-AR Models - Part II
* 1.7 Non-AR Models - Part III
* 1.8 Skinny Models I
* 1.9 Skinny Models II

> Our controllers are kind of chubby, so let's make things better.
Combine the two conditionals that involve validating and creating a
review into a single conditional that calls@review.add_to_item and returns a boolean.
Don't worry, we'll implement this method next.

```ruby
class ReviewsController < ApplicationController
  def create
    @item = Item.find(params[:review][:item_id])
    @review = @item.reviews.build(review_params)
    # start refactoring here...
    if @review.bad_words?
      flash[:error] = 'Did not save review.'
      render :new
    elsif @review.save
      redirect_to @review, notice: 'Successfully created'
    else
      flash[:error] = 'Did not save review.'
      render :new
    end
  end

  def new
    @review = Review.new
  end

  def show
    @review = Review.find(review_params)
  end

  private

  def review_params
    params.require(:review).permit(:description)
  end
end

class Review < ActiveRecord::Base
  belongs_to :item

  def add_to_item
    # We'll fill this in next!
  end

  private

  def bad_words?
    description =~ /BAD_WORD/
  end
end

  # 重构为
class ReviewsController < ApplicationController
  def create
    @item = Item.find(params[:review][:item_id])
    @review = @item.reviews.build(review_params)
    # start refactoring here...
    if @review.add_to_item
      redirect_to @review, notice: 'Successfully created'
    else
      flash[:error] = 'Did not save review.'
      render :new
    end
  end

  def new
    @review = Review.new
  end

  def show
    @review = Review.find(review_params)
  end

  private

  def review_params
    params.require(:review).permit(:description)
  end
end
```


> Now let's implement the add_to_item method.
We want to call save only if bad_words? returns false. (Using a guard clause can help here..)

```ruby
class Review < ActiveRecord::Base
  belongs_to :item

  def add_to_item
    # Your code goes here
  end

  private
    def bad_words?
      description =~ /BAD_WORD/
    end
end

  # 重构为
class Review < ActiveRecord::Base
  belongs_to :item

  def add_to_item
    if bad_words?
      return false
    end

    save
  end

  private
    def bad_words?
      description =~ /BAD_WORD/
    end

end
```


> Notice how inside our ItemsController create action we're setting the item.pretty_url attribute.
This would be a good method to place inside our model as a callback so it would happen every time the model is saved.
Create a set_pretty_url method inside of the Item model and use a before_save callback to call it whenever the model is saved.
Make sure the set_pretty_url method is protected so it doesn't accidentally get used outside of the callback.

```ruby
class Item < ActiveRecord::Base

end

  # 重构为
class Item < ActiveRecord::Base
  before_save :set_pretty_url

  protected
    def set_pretty_url
      self.pretty_url = name.parameterize
    end
end
```

> Let's extract some registration logic out of our controllers into a UserRegistration class.
This class should take user_params as arguments to its constructor, which are used to initialize a new User (not create).
This newly initialized user should be available as an attr_reader.
You'll also want to move the valid_background_check? method into this new class as a private method, we'll use this later to finish creating the User.

```ruby
class UserRegistration

  private
  # private methods go here
end

  # 重构为
class UserRegistration
  # Add an `attr_accessor` for :user
  attr_reader :user

  # Initialize a new User in the constructor
  def initialize(params)
    @user = User.new(params)
  end

  private

  def valid_background_check?
    !!(@user.valid_ssn? || @user.valid_address?)
  end
end
```


# Now let's implement the #create method.
# First, we need to set user.is_approved to true if valid_background_check? returns true.
# Then we can call user.save to finish creating the user.


class UserRegistration
  attr_reader :user

  def initialize(params)
    @user = User.new(params)
  end

  private

  def valid_background_check?
    !!(@user.valid_ssn? || @user.valid_address?)
  end
end

##-->
class UserRegistration
  attr_reader :user

  def initialize(params)
    @user = User.new(params)
  end

  def create
    if valid_background_check?
      user.is_approved = true
    end

    user.save
  end

  private

  def valid_background_check?
    !!(@user.valid_ssn? || @user.valid_address?)
  end
end


# Great! Now that you have completed the UserRegistration class it's time to change the UsersController to use it.
# Make sure to remove the valid_background_check? method, since you moved that into the registration class.


class UsersController < ApplicationController
  def create
    @user = User.new(user_params)

    if valid_background_check?
      @user.is_approved = true
    end

    if @user.save
      redirect_to @user
    else
      render :new
    end
  end

  private

  def valid_background_check?
    !!(@user.valid_ssn? || @user.valid_address?)
  end

  def user_params
    params.require(:user).permit(:name, :email, :ssn, :address)
  end
end

##-->
class UsersController < ApplicationController
  def create
    registration = UserRegistration.new(user_params)
    @user = registration.user

    if registration.create
      redirect_to @user
    else
      render :new
    end
  end

  private

  def user_params
    params.require(:user).permit(:name, :email, :ssn, :address)
  end
end

# Let's take a look at the welcome method of the User model.
# This method is doing way too many things, you really need to split it up into separate private methods.
# Create the following private methods: send_welcome_email, enable_welcome_tour, enable_welcome_promotion.
# Then move the code from the welcome method into each one.
# Make sure to still call each method from within the welcome method.

class User < ActiveRecord::Base
  def welcome
    # send_welcome_email
    WelcomeMailer.welcome(self).deliver

    # enable_welcome_tour
    self.welcome_tour = true
    self.save

    # enable_welcome_promotion
    promo = Promotion.new(name: "Thanks for joining!")
    promo.set_redeemer(self)
  end
end


##-->
class User < ActiveRecord::Base
  def welcome
    send_welcome_email
    enable_welcome_tour
    enable_welcome_promotion
  end

  private

  def send_welcome_email
    WelcomeMailer.welcome(self).deliver
  end

  def enable_welcome_tour
    self.welcome_tour = true
    self.save
  end

  def enable_welcome_promotion
    promo = Promotion.new(name: "Thanks for joining!")
    promo.set_redeemer(self)
  end
end

# That looks better.
# You can take this a step further by extracting out the welcome user functionality into a separate class similar to the UserRegistration class you created a few challenges back.
# This new class should accept an instance of User as an argument for the constructor.
# It should also have an attr_accessor for the user and a welcome method which functions the same as the original welcome method.

class UserWelcome

end

##-->
class UserWelcome
  # Add an `attr_accessor` for :user
  attr_accessor :user

  # Create the constructor and save the user to a @user instance variable
  def initialize(user)
    @user = user
  end

  # Bring over the `welcome` method from the User class and it's associated methods.
  # Replace `self` with `user` or `@user`
  def welcome
    send_welcome_email
    enable_welcome_tour
    enable_welcome_promotion
  end

  private

  def send_welcome_email
    WelcomeMailer.welcome(user).deliver
  end

  def enable_welcome_tour
    user.welcome_tour = true
    user.save
  end

  def enable_welcome_promotion
    promo = Promotion.new(name: "Thanks for joining!")
    promo.set_redeemer(user)
  end
end


