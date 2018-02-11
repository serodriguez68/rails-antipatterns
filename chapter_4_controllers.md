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
    * Rails Anti-patterns refers to this as Presenters.
    * Code Climate refers to this as Form Objects. 
    * Rails Casts refers to this as Form Objects.
    * Go Rails refers to this as From Objects.

* Decorators (as explained by [Code Climate in this post](https://codeclimate.com/blog/7-ways-to-decompose-fat-activerecord-models/)): They are an alternative to using callbacks in models. Decorator objects should quack like the original object but can tap into some of it's methods to modify the behavior. Apparently, this type of decorators are not so popular in the Rails community and are replaced by service objects.

* __Service Objects__: wrapper object that orchestrate multiple actions involving several active record objects. The community has many different ways of implementing service objects.
    *  [Interactor](https://github.com/collectiveidea/interactor) is an opinionated library for service objects.
    * [The use case pattern](https://webuild.envato.com/blog/a-case-for-use-cases/) introduced by Envato comes with a [small library](https://github.com/stevehodgkiss/interaction).
    *  [Here](https://medium.com/selleo/essential-rubyonrails-patterns-part-1-service-objects-1af9f9573ca1) are some general good practices for service objects.


### 4.2.2 Problem: Using hand-rolled auxiliary save or hand-roll methods in models to create associated records of an instance.

```ruby
class ArticlesController < ApplicationController 
    def create
        @article = Article.new(params[:article])
        @article.reporter_id = current_user.id
        begin 
            Article.transaction do
                @version = @article.create_version!(params[:version], current_user)
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

YOU ARE HERE
