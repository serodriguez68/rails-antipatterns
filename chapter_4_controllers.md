# Chapter 4 - Controllers
## 4.1 Anti-Pattern: Homemade Authentication
### Solution: Use a well-known library that suits your needs
DON'T make a homemade authentication system unless you are under extreme circumstances.

Here are some names of popular authentication libraries:

* [Devise](https://github.com/plataformatec/devise): Fully featured authentication gem (probably the most well-known gem).
* [Clearance](https://github.com/thoughtbot/clearance): In the middle solution for authentication.
* [Authlogic](https://github.com/binarylogic/authlogic): Minimalistic authentication solution that pushes down the authentication logic to models.

## 4.2 Anti-Pattern: Fat Controller
### 4.2.1 Problem: Using hand-rolled auxiliary save or create methods in models to create associated records of an instance.

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

>Notes form the summarizer: `callbacks` and `accepts_nested_attributes_for` are __very controversial__ topics.  With experience you will form an opinion on how to use them in __moderation__.  One sign that your callbacks are going the wrong way are slow tests, brittle tests, or an urge to stub out all callback side effects for wholly unrelated tests.
