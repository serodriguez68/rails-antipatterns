# Chapter 4 - Controllers
## 4.1 Anti-Pattern: Homemade Authentication
### Solution: Use a well-known library that suits your needs
DON'T make a homemade authentication system unless you are under extreme circumstances.

Here are some names of popular authentication libraries:

* [Devise](https://github.com/plataformatec/devise): Fully featured authentication gem (probably the most well-known gem).
* [Clearance](https://github.com/thoughtbot/clearance): In the middle solution for authentication.
* [Authlogic](https://github.com/binarylogic/authlogic): Minimalistic authentication solution that pushes down the authentication logic to models.

## 4.2 Anti-Pattern: Fat Controller

### 4.2.1 Important Notes About Pattern Names in the Rails Community
Pattern naming  is apparently very inconsistent in the Rails community. You will find the same name used for different patterns, which makes everything very confusing.

Here are my 2 cents to try to clear up the confusion:

#### Patterns to Fix Views:

* __Decorator / Presenter / View Models / View Objects:__ Create objects that "wrap" models and contain only view specific logic.  [Draper](https://github.com/drapergem/draper) is a popular gem for doing this.
    * Rails Casts refers to this as Presenters.
    * GoRails refers to this as Decorators
    * Code Climate refers to this as View Objects.
    * Draper refers to this as Decorators.

#### Patterns to re-factor Fat Controllers (and also Avoiding Fat Models):

* __Presenter / Form Object:__ When multiple Active Record objects are updated in a single form, a form object can be used to wrap the manipulation of multiple models into one.  The __form object__ quacks like an Active Record object, so all your controller code keeps familiar.
    * [Reform](https://github.com/trailblazer/reform) is the most popular form object gem for Ruby. 
    * Rails Anti-patterns refers to this as Presenters.
    * Code Climate refers to this as Form Objects. 
    * Rails Casts refers to this as Form Objects.
    * Go Rails refers to this as From Objects.

* Decorators (as explained by [Code Climate in this post](https://codeclimate.com/blog/7-ways-to-decompose-fat-activerecord-models/)): They are an alternative to using callbacks in models. Decorator objects should quack like the original object but can tap into some of it's methods to modify the behavior. Apparently, this type of decorators are not so popular in the Rails community and are replaceable by service objects.  Because of this, Decorators are not included in this summary.

* __Service Objects__: The term "Service Object" is used to refer to wildly different code implementations.  There seem to exist 2 broad types of form objects:
    * __Service Objects that coordinate sequences of steps__
        * __Enhanced form objects__: it is fairly common to see posts about form objects that also have the responsibility to coordinate steps before, during and after the persistence. Examples of these posts are
            *   [ThoughtBot Approach](https://robots.thoughtbot.com/activemodel-form-objects)
            *   [Envato Approach](https://webuild.envato.com/blog/creating-form-objects-with-activemodel-and-virtus/)
        * __Other approaches__   
            *  [Interactor](https://github.com/collectiveidea/interactor) is an opinionated library for service objects.
            * [The use case pattern](https://webuild.envato.com/blog/a-case-for-use-cases/) introduced by Envato comes with a [small library](https://github.com/stevehodgkiss/interaction).
            *  [Here](https://medium.com/selleo/essential-rubyonrails-patterns-part-1-service-objects-1af9f9573ca1) are some general good practices for service objects.

    
    * __Service Objects that provide logic for a very specific part of your app__
        * Examples: a PORO that wraps an API is a service object. 
        
* __Changes in Rails Architecture: [Trailblazer](http://trailblazer.to/)__
    * Trailblazer proposes a different architecture to solve the problem of complex business logic that makes your controllers and/or models Fat.
        * Trailblazer is big and complex.  It deserves a summary on it's own. 

### 4.2.2 Problem: Using hand-rolled auxiliary save or hand-roll methods in models to create associated records of an instance.

```ruby
class ArticlesController < ApplicationController 
    def create
        @article = Article.new(params[:article])
        @article.reporter_id = current_user.id
        begin 
            Article.transaction do
                @version = @article.create_version!(params[:version], current_user)  # This is the hand-rolled auxiliary method
            end
        rescue ActiveRecord::RecordNotSaved, ActiveRecord::RecordInvalid
            render :index and return false
        end
        redirect_to article_path(@article)
    end
end
```

The problems with this code are:

* The controller action is too long. 14 lines of code is not much but it is certainly on the limit of length for a controller action.
* It is defining a database transaction block.  Controllers should never define DB transaction blocks (this is an indication that it is doing too much).
    * Take into account that normal active record life-cycle methods such as `save` and callbacks are automatically wrapped in transaction blocks. 
* The `@article` is never explicitly `saved`. That is very confusing.
* Use of `Exceptions` to control flow.  Exceptions should not be used to control flow.
* `create_version!` is a hand-rolled life-cycle method.  Strive for using only the default life-cycle methods (e.g `save`, `create`).
* `@article.reporter_id = current_user.id` is bad style. `@article.reporter = current_user` is preferred.


#### Solution: Use Active Record Callbacks and Nested Attributes
>The re-factor process shown by the authors is long and difficult to move into this summary. Please refer to the book.  Additionally, despite being classified by the authors as a __controller__ anti-pattern, the re-factor mainly involves modifications to the `Article` and `Version` models. Nevertheless, I registered the main techniques in this summary.

The resulting controller action after re-factor is:

```ruby
class ArticlesController < ApplicationController 
    def create
        @article = Article.new(params[:article]) 
        @article.reporter = current_user 
        @article.new_version.writer = current_user
    
        if @article.save
            render :index
        else
            redirect_to article_path(@article)
        end 
    end
end
```

The main refactoring points were:

* The custom saving method `create_version!` was eliminated from the `Article`model.  This required the use of 2 techniques:
    * Use of `callbacks` on the `Version` model to handle all logic related to bumping article versions, deprecating old versions, tagging the current version on the `article` instance (that has many versions).
    * Use of `nested attributes` to be able to create a `version` instance through `Article.new(params[:article])` (by passing in params of the shape `params[:article][:new_version]`.
* Change the style of associations like `@article.reporter_id = current_user.id` to `@article.reporter = current_user`.
* Get rid of exceptions for control flow and use the regular boolean value returned by the `save` method.

>Notes form the summarizer: `callbacks` and `accepts_nested_attributes_for` are __very controversial__ topics.  With experience you will form an opinion on how to use them in __moderation__.  As a rule of thumb moving manipulation of multiple models into callbacks is only OK if the functionality is simple AND the manipulation of associated models can be viewed as the responsibility of the primary model.

>One sign that your callbacks are going the wrong way are slow tests, brittle tests, or an urge to stub out all callback side effects for wholly unrelated tests.

### 4.2.3 Problem: 1 form contains fields of different models and the controller ended up manipulating multiple objects.

The scenario is this: In order for your users to sign up, we need to create both a `User` and an `Account` record in the database. We need to make sure that either both or none are created, hence we have to use database transactions.

The resulting code looks like this:
```ruby
class AccountsController < ApplicationController
    def new
        @account = Account.new 
        @user = User.new
    end

    def create
        @account = Account.new(params[:account])
        @user = User.new(params[:user])
        @user.account = @account
        
        ActiveRecord::Base.transaction do @account.save!
            @account.save!
            @user.save!
        end

        flash[:notice] = 'Account was successfully created.' 
        redirect_to(@account)
    rescue ActiveRecord::RecordInvalid, ActiveRecord::RecordNotSaved 
        render :action => "new"
    end 
end
```

Problems with this code:
* Using Exceptions to control flow
* Introducing DB transactions, a low-level database concept, into the controller layer.
    * Rule of thumb: if you controller is doing explicit use of transactions, you are going the wrong way.
* The names `AccountsController#create` and `AccountsController#new` are misleading since they also manipulate users.

#### Solution: Use the Form Object Pattern
>Another indication that you may be needing a form object: You have a model (e.g `User`) that is updated thorough multiple forms that look and behave differently (e.g a sign up form behaves differently than a reset password form).

A form object is typically a PORO that quacks like an Active Record object but under the hood it manipulates multiple models in your database, saving them in a coordinated manner and (possibly) performing some steps in between.

>* Performing additional steps in between write operations in a form object is fine if the logic is simple. Once the logic starts to get complex, consider using service objects to encapsulate specific logic units.

Form objects also provide a way of performing _contextual validations_. Sometimes it is desirable to trigger certain validations when the user is filling a specific form.  For example we want to force users to input their social media profiles when they are filling the _"complete your profile"_ from BUT NOT when they are signing up.  If this validation was written down in the model level, we would need to resort to tricks to make sure the validation is enforced on certain circumstances.

There are many ways of implementing form objects. I will show 2 ways, but here is a list of some of the popular ways to do it:

* Using the `Active Presenter` gem (shown - super simple solution).
* Hand-rolling your solution with `Active Model` (shown - intermediate solution).
* Using the [reform gem](https://github.com/trailblazer/reform) (NOT shown - advanced solution).

##### Form Objects with [`Active Presenter`](https://github.com/jamesgolick/active_presenter)
```ruby
# app/models/signup.rb
class Signup < ActivePresenter::Base
    before_save :assign_user_to_account 
    presents :user, :account
private
    def assign_user_to_account 
        user.account = account
    end 
end
```
Some important points about `Active Presenter`:

* `Active Presenter` maps the fields on the models to the presenter by prepending the model name to each field (e.g User#email becomes Signup#user_email).
* `presents :user, :account` indicates that this presenter deals with `user` and `account` objects.
* The `Presenter` automagically inherits the errors from the presented objects based on their validations.  This means that we don't need to do anything new to use the available __model level validations__.
    * However this also means that we can't have _contextual validations_. (See hand-rolled version for example on contextual validations.) 

Given that `Signup` quacks like an active record object, the views and controllers remain consistent with the "Rails Way":

```erb
# app/views/signups/new.html.erb
<h1>Signup!</h1>
<%= form_for(@signup) do |f| %> 
    <%= f.error_messages %>
    <p>
        <%= f.label :account_subdomain %><br /> 
        <%= f.text_field :account_subdomain %>
    </p>
    <p>
        <%= f.label :user_email %><br /> 
        <%= f.text_field :user_email %>
    </p>
    <p>
        <%= f.label :user_password %><br /> 
        <%= f.text_field :user_password %>
    </p>
    <p>
        <%= f.submit "Create" %>
    </p> 
<% end %>
```

```ruby
# app/controllers/signup_controller.rb
class SignupsController < ApplicationController
    def new
        @signup = Signup.new
    end

    def create
        @signup = Signup.new(params[:signup])
        if @signup.save
            flash[:notice] = 'Thank you for signing up!' redirect_to root_url
        else
            render  "new"
        end 
    end
end
```

##### Hand-rolled Form Objects with `Active Model`
Hand rolling your own form objects is very simple if you use `Active Model`. One thing to keep in mind is that you will probably need to abandon the idea of doing validations in the model and move all validation responsibility to the form objects (or duplicating it).

Here are 2 execellent posts on how to hand-roll your own form objects:

* [ThoughtBot Approach](https://robots.thoughtbot.com/activemodel-form-objects)

* [Envato Approach](https://webuild.envato.com/blog/creating-form-objects-with-activemodel-and-virtus/)
