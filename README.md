# Rails Anti-patterns Summary
Notes by: __[Sergio Rodriguez](https://github.com/serodriguez68 "Sergio's Github")__ 

__These are some notes I took while reading the book. Feel free to send me a pull request if you want to make an improvement.__
_______________________________________________________________________________

# Chapter 1 - Models
## 1.1 Anti-pattern: Voyeuristic Models
### 1.1.1 Problem: Traversing too far through associations (Law of Demeter Violation)

Consider the following case: 
In your e-commerce app Customers only have one Address and Customers can have many Invoices:
```ruby
class Address < ActiveRecord::Base 
    belongs_to :customer
end

class Customer < ActiveRecord::Base 
    has_one :address
    has_many :invoices
end

class Invoice < ActiveRecord::Base 
    belongs_to :customer
end
```

In several views you have this kind of code to show details of an invoice: 

```erb
<%= @invoice.customer.name %>
<%= @invoice.customer.address.street %>
<%= @invoice.customer.address.city %>
<%= @invoice.customer.address.state %>
<%= @invoice.customer.address.zip_code %>
```

__Breaking change:__ you are now asked to implement the concept of _billing address_ and _shipping address_.  This means that you need to change all of the views to account for that change.

#### Solution 1: Use wrapper methods
Wrapper methods allow us to ask the __direct__ neighbors for information.  If you are asking for information that is not part of you direct neighbor's attributes, it is __their__ responsibility to ask __their direct__ neighbors for the information.

```ruby
class Address < ActiveRecord::Base 
    belongs_to :customer
end

class Customer < ActiveRecord::Base 
    has_one :address
    has_many :invoices
    
    def street   address.street   end
    def city     address.city     end
    def state    address.state    end
    def zip_code address.zip_code end
end

class Invoice < ActiveRecord::Base 
    belongs_to :customer
    def customer_name     customer.name     end
    def customer_street   customer.street   end
    def customer_city     customer.city     end
    def customer_state    customer.state    end
    def customer_zip_code customer.zip_code end
end
```

You could change the view code to the following:

```erb
<%= @invoice.customer_name %> 
<%= @invoice.customer_street %> 
<%= @invoice.customer_city %>
<%= @invoice.customer_state %>
<%= @invoice.customer_zip_code %>
```

__Cons of this approach:__ your public interface's code gets murky with many methods that are arguably non related to you class (e.g Invoice#customer_state)

#### Solution 2: Use [delegate](http://api.rubyonrails.org/classes/Module.html#method-i-delegate)

__Note:__ Delegation should only be used to talk do __direct neighbors.__ See [use wrapper methods](#solution-1-use-wrapper-methods) for more information.

```ruby
class Address < ActiveRecord::Base  
    belongs_to :customer
end
class Customer < ActiveRecord::Base 
    has_one :address
    has_many :invoices
    delegate :street, :city, :state, :zip_code, to: :address
end
class Invoice < ActiveRecord::Base 
    belongs_to :customer
    delegate :name, :street, :city, :state, :zip_code, to: :customer, prefix: true
end
```
The view code remains the same as in solution 1.

### 1.1.2 Problem: Defining queries in the Views or Controllers
```erb
<ul>
    <% User.find(order: "last_name").each do |user| -%>
        <li><%= user.last_name %> <%= user.first_name %></li>
    <% end %>
</ul> 
```
This is a problem because you are most likely repeating this code on all views that render a list of users.

Pushing down this to a controller is a bit of an improvement. However, most likely you are repeating the code on all controller actions that require a list of users. 
Additionally, if you want to change you default user ordering you want to be able to change it __everywhere__ from a single place.

```ruby
class UsersController < ApplicationController 
    def index
        @users = User.order("last_name") 
    end
end
```

#### Solution: Push all queries down to the Models as class methods or named scopes
```ruby
class User < ActiveRecord::Base 
    def self.ordered 
        order("last_name") 
    end

    # OR AS A NAMED SCOPE
    scope :ordered, -> { order("last_name") } 
end
```


### 1.1.3 Problem: Scopes/Finders located on a model include finders from other models

Consider the following code
```ruby
class User < ActiveRecord::Base 
    has_many :memberships
    def find_recent_active_memberships 
        memberships.where(:active => true).limit(5)
        . order("last_active_on DESC")
    end 
end
```
In this case, `find_recent_active_memberships` knows too much about what an active membership is and how are they supposed to be ordered.  This makes `User` know too much about the `Membership` model.

#### Solution: Move the scopes to the appropriate models
After moving the scopes you must call the __neighboring__ model's scope instead of repeating the query.

Here are 2 alternatives:
```ruby
# Alternative 1
class User < ActiveRecord::Base 
    has_many :memberships
    def find_recent_active_memberships 
        memberships.find_recently_active
    end 
end

class Membership < ActiveRecord::Base 
    belongs_to :user
    def self.find_recently_active
        where(:active => true).limit(5).order("last_active_on DESC")
    end 
end
```

```ruby
# Alternative 2
class User < ActiveRecord::Base 
    has_many :memberships
    def find_recent_active_memberships 
        memberships.only_active.order_by_activity.limit(5)
    end 
end

class Membership < ActiveRecord::Base 
    belongs_to :user
    scope :only_active, where(:active => true)
    scope :order_by_activity, order('last_active_on DESC') 
end
```

## 1.2 Anti-pattern: Fat Models
### 1.2.1 Problem: A Model Has Grown Beyond It's Purpose
Classes must have only __one__ reason to change. If a class has more than one reason to change, that means it is breaking the single responsibility principle.

Consider the following example:
```ruby
class Order < ActiveRecord::Base
    def self.find_purchased ... end
    def self.find_waiting_for_review ... end
    def self.find_waiting_for_sign_off ... end
    def self.advanced_search(fields, options = {}) ... end
    def self.simple_search(terms) ... end
    def to_xml ... end
    def to_json ... end
    def to_csv ... end
    def to_pdf ... end
end
```
This order class has the responsibility of finding orders and also the responsibility of converting them to multiple formats, breaking the single responsibility principle.

#### Solution 1: Delegate Responsibility to New Classes using hand-rolled composition
Move all the conversion logic to it's own class and let the `Order` class deal with _order like_ logic only.

```ruby
class OrderConverter
    attr_reader :order
    def initialize(order) @order = order end
    def to_xml ... end
    def to_json ... end
    def to_csv ... end
    def to_pdf ... end
end
```
Now you can _compose_ the `OrderConverter` object into the `Order` object itself.

```ruby
class Order < ActiveRecord::Base
    # This is composition.  Order is composed of OrderConverter
    def converter OrderConverter.new(self) end 
end
```
_Note:_ The Rails association methods (for example, has_one, has_many, belongs_to) all create this sort of composition automatically for database-backed models.

At this point you can do `@order.converter.to_pdf`. However, note that this breaks the [Law of Demeter](#111-problem-traversing-too-far-through-associations-law-of-demeter-violation). [Hence, we can use delegation to fix the problem](#solution-2-use-deleagate).

```ruby
class Order < ActiveRecord::Base
    delegate :to_xml, :to_json, :to_csv, :to_pdf, to: :converter
    def converter OrderConverter.new(self) end 
end
```

#### Solution 2: Delegate Responsibility to New Classes using Rails `composed_of`
The underlying idea is the same.  However, we will use ActiveRecord to make the code a little less verbose.

We will use another example to illustrate this solution.

__Problematic Code__
```ruby
class BankAccount < ActiveRecord::Base
    validates :balance_in_cents, :presence => true 
    validates :currency, :presence => true
    # Bank account logic...transfer, deposit, withdraw, open, close.
    
    # Money-Like logic
    def balance_in_other_currency(currency) currency exchange logic... end
    def balance balance_in_cents / 100 end
    def balance_equal?(other_bank_account) 
        balance_in_cents == 
            other_bank_account.balance_in_other_currency(currency) 
    end
end
```
The problem here is that `BankAccount` takes both the responsibility of being an account and also the responsibility of money (currency exchange, comparing against other money, etc).

> `composed_of` takes 3 main options: 1) name of the method that will reference the new object, 2) the name of the object's class, and the mapping of database columns to attributes of the object.

This is the `composed_of` solution.
```ruby
class BankAccount < ActiveRecord::Base
    validates :balance_in_cents, :presence => true 
    validates :currency, :presence => true
    composed_of :balance,
                class_name: "Money",
                mapping: [%w(balance_in_cents amount_in_cents),
                          %w(currency currency)]
end

# app/models/money.rb 
class Money
    include Comparable
    attr_accessor :amount_in_cents, :currency

    def initialize(amount_in_cents, currency) 
        self.amount_in_cents = amount_in_cents 
        self.currency = currency
    end

    def in_currency(other_currency) currency exchange logic... end
    def amount amount_in_cents / 100 end

    def <=>(other_money) 
        amount_in_cents <=> other_money.in_currency(currency).amount_in_cents 
    end
end
```
Now you can: `@bank_account.balance.in_currency(:usd)` and `@bank_account.balance > @other_bank_account.balance`.

#### Not So Great Solution 3: Moving methods into modules

Extracting the extra responsibility methods in a module that is only included in that class is NOT the way to go.  This helps organizing the code a bit, and reduces visual complexity.  However, the class still has more than one responsibility.

Here is some code to illustrate how this is done.
```ruby
# This is the problematic code
class Order < ActiveRecord::Base
    # Find Orders by State
    def self.find_purchased ... end
    def self.find_waiting_for_review ... end
    def self.find_waiting_for_sign_off ... end
    
    # General Order Searchers
    def self.advanced_search(fields, options = {}) ... end
    def self.simple_search(terms) ... end
    
    # Order Converters
    def to_xml ... end
    def to_json ... end
    def to_csv ... end
    def to_pdf ... end
end
```

In the _not-so-great solution_ we identify that all methods inside Order can be grouped in three buckets.  We create modules for each bucket and the include/extend them inside order.

```ruby
# This is the not-so-great solution code
class Order < ActiveRecord::Base
    extend OrderStateFinders 
    extend OrderSearchers 
    include OrderExporters
end

# lib/order_state_finders.rb 
# Omitted for simplicity

# lib/order_searchers.rb  or in Rails 4+ could be somewhere in concerns
module OrderSearchers
    def advanced_search(fields, options = {}) ... end
    def simple_search(terms) ... end 
end

# lib/order_exporters.rb or in Rails 4+ could be somewhere in concerns
module OrderExporters
    def to_xml ... end
    def to_json ... end
    def to_csv ... end
    def to_pdf ... end
end
```

Always try to favor _composition_ over _inheritance_.  Inclusion of modules is a type of inheritance.

### 1.2.2 Problem: Custom methods in models for sequential manipulation of multiple models
__(and the large transaction blocks that come with them)__

Consider the following code:

```ruby
class Account < ActiveRecord::Base
    def create_account!(account_params, user_params)
        transaction do
            account = Account.create!(account_params)
            first_user = User.new(user_params)
            first_user.admin = true
            first_user.save!
            self.users << first_user
            account.save! 
            Mailer.deliver_confirmation(first_user) 
            return account
        end 
    end
end
```

There 2 main problems with this code:

* The Account class has the responsibility of manipulating other classes  (`User` and `Mailer`) through the  `create_account!` method. (This is the main problem).
* The `transaction` block can be avoided by using standard Rails validations and callbacks.

> __Note from the summarizer:__ I agree that this is an anti-pattern and that there are better ways for handling this type of code.  However, I don't  agree with the book's suggested solution.  I include that solution, for the sake of completeness. Nevertheless, take the time to take a look at things like [the use case pattern](https://webuild.envato.com/blog/a-case-for-use-cases/) or service objects, I believe they are much more elegant solutions to the main problem (even if you can't avoid the transaction block).

#### Proposed Solution: fixing the call to `User` by using `nested_attributes` and send email with a callback

__Part 1: using nested_attributes__
All the hand-rolled manipulation that we are doing to the `User` model can be covered by Rails' `nested_attributes`.  

By adding `accepts_nested_attributes_for :users` to `Account`, the Account model will now be able to manipulate `User`  when `Account#new, Account#create, Account#update_attributes` are called with a `user_attributes` subhash.

This in turn means that we need to modify our view, to make the form send the information with the `user_attributes` subhash in the proper format.

```ruby
<%= form_for(@account) do |form| -%>
    <%= form.label :name, 'Account name' %>
    <%= form.text_field :name %>
    <% fields_for :user, User.new do |user_form| -%>
        <%= user_form.label :name, 'User name' %>
        <%= user_form.text_field :name %>
        <%= user_form.label :email %>
        <%= user_form.text_field :email %>
        <%= user_form.label :password %>
        <%= user_form.password_field :password %>
    <% end %>
    <%= form.submit 'Create', :disable_with => 'Please wait...' %>
<% end %>
```

At this point your `Account` class should look like this.  Note that you still need to make the user an admin, hence the `make_admin_user` callback.

```ruby
class Account < ActiveRecord::Base
    accepts_nested_attributes_for :users
    before_create :make_admin_user
    private
        def make_admin_user 
            self.users.first.admin = true
        end
    # ...
end
```

> __Note from the summarizer:__ I personally think that `nested_attributes` just hides a bad object oriented programming practice.  A proof of that is that we ended up implementing a `make_admin_user` callback that manipulates the user from the account (And that is not even talking about the problems of callbacks themselves).  A smarter approach could be using a [form object pattern](https://codeclimate.com/blog/7-ways-to-decompose-fat-activerecord-models/).

__Part 2: Using an `after_create` callback to send the email__
The final proposed solution is the following:

```ruby
class Account < ActiveRecord::Base 
    accepts_nested_attributes_for :users
    before_create :make_admin_user 
    after_create :send_confirmation_email
    private
        def make_admin_user 
            self.users.first.admin = true
        end
        def send_confirmation_email 
            Mailer.confirmation(users.first).deliver
        end
    # ...
end
```
> __Note from the summarizer:__ I don't agree with sending the email through a callback. Callbacks give your classes a bunch of unexpected and __very hard to debug__ side effects that you are better-off avoiding. Imagine you now have a face to face account creation process that does not need an email.  Bad luck, with a callback __every creation__ will send an email and you can't avoid that.

> __Note from the summarizer:__ The result of the proposed refactor is a very small Account class.  In my opinion this is at the expense of hiding a lot of complexity behind `nested_attributes` and paying a high cost in committing to __always__ send an email on creation (callback).

## 1.3 Anti-pattern: Spaghetti SQL
### 1.3.1 Problem: Active Record associations are not leveraged

Consider the following code:
```ruby
class PetsController < ApplicationController 
    def show
        @pet = Pet.find(params[:id])
        @toys = Toy.where(:pet_id => @pet.id, :cute => true)
    end
end
```
This code has 2 problems:
* The controller is aware of the Toy implementation (it knows what columns does Toy have).
* The association from pet to toys is not being used.

#### Solution 1: Define scopes on the responsible models and use associations
__90% of the times you would use this solution. Other solutions are shown below, but they are useful in very specific cases only.__

```ruby
class PetsController < ApplicationController 
    def show
        @pet = Pet.find(params[:id])
        @toys = @pet.toys.cute.paginate(params[:page]) 
    end
end

class Toy < ActiveRecord::Base 
    scope :cute, -> { where(:cute => true) } 
end

class Pet < ActiveRecord::Base 
    has_many :toys
end
```
Notice that:
* The `Toy` class is the one responsible for defining what `#cute` means.
* The controller leverages the association.

#### Solution 2: `extend` the association using Modules or Blocks
Rails allows us to "enrich" a declared association with methods by passing in a block OR extending a module.  This gives access to the _proxy owner_ (the original caller) inside the method.

Although this is rarely needed, is can be very handy __under certain conditions__. The following code shows an example when this type of "association enrichment" is the way to go.

```ruby
class Toy < ActiveRecord::Base 
    # has column :minimum_age
end
class Pet < ActiveRecord::Base 
    # has column :age
    
    # The association :toys is being enriched by passing a block.
    has_many :toys do 
        def appropriate
            where(["minimum_age < ?", proxy_owner.age]) 
        end
    end 
end
```

This association enrichment now allows to do `pet.toys.appropriate`.  Note that `.appropriate` makes use of the __pet's age__ (the caller / proxy owner) inside the scope.

The syntax for association enrichment with modules is the following. A different and non-coherent example is used to illustrate the syntax:
```ruby
module ToyAssocationMethods 
    def cute
        where(:cute => true) 
    end
end
class Pet < ActiveRecord::Base
    has_many :toys, :extend => ToyAssocationMethods
end
```

### 1.3.2 Problem: Complex finders have complicated non-modular code
Consider this example for searching songs according to multiple parameters.
```ruby
class Song < ActiveRecord::Base
    def self.search(title, artist, genre, published, order, limit, page)
        conditional_values = { title: "%#{title}%", 
                               artist: "%#{artist}%",
                               genre: "%#{genre}%" }
        case order
        when "name" : order_clause = "name DESC"
        when "length" : order_clause = "length ASC"
        when "genre": order_clause = "genre DESC" 
        else
            order_clause = "album DESC" 
        end

        joins = []
        conditions = []
        conditions << "(title LIKE ':title')" unless title.blank?
        conditions << "(artist LIKE ':artist')" unless artist.blank?
        conditions << "(genre LIKE ':genre')" unless genre.blank?

        unless published.blank?
            conditions << "(published_on == :true OR published_on IS NOT NULL)"
        end 

        find_opts = { conditions: [conditions.join(" AND "), condition_values],
                      joins: joins.join(' '),
                      limit: limit,
                      order: order_clause } 
        
        
        page = 1 if page.blank?
        paginate(:all, find_opts.merge(:page => page, :per_page => 25))
    end
end
```
This code is a huge mess:
* It implements multiple searches in one (e.g by title, by artist).
* It combines the responsibility of searching, with the responsibility of ordering and paginating.
* Paginating is NOT a responsibility of the model (normally the controller does this).

#### Solution: Break Method Into Modular Scopes
The solution does this:
* Abstracts repeated finders into a generic `matching` method.
* Separates finders, ordering and limiting by implementing separate methods for each responsibility. 
* Builds a robust `search` that is made up of simple components.
 
```ruby
class Song < ActiveRecord::Base
    scope :top, -> (number) { limit(number) }
    scope :matching, -> (column, value) { 
        where(["#{column} like ?", "%#{value}%"]) 
    }
    scope :published, -> { where.not(published: nil) }
    
    # Note from the summarizer:
    # I am not a big fan of this implementation.  However,
    # the point being illustrated is to separate the ordering form the finding.
    def self.order(col)
        sql = case order
                when "name" : order_clause = "name DESC"
                when "length" : order_clause = "length ASC"
                when "genre": order_clause = "genre DESC" 
                else order_clause = "album DESC" 
              end
        order(sql)
    end

    def self.search(title, artist, genre, published)
        finder = matching(:title, title)
        finder = finder.matching(:artist, artist)
        finder = finder.matching(:genre, genre)
        finder = finder.published unless published.blank? return finder
    end 
end

# Callers can use the search method like this:
Song.search("fool", "billy", "rock", true).
     order("length").
     top(10).
      paginate(:page => 1)
```

### 1.3.3 Problem: Hand-rolled Text Search Goes Out of Control
Full text search is a complicated problem period.  For simple searches you may be able to survive with your database's `LIKE` search functionality.  However, once you start doing fuzzy search in multiple columns at a time, the code gets really ugly really fast.

Here is an example of a hand-rolled search functionality that became a mess:
```ruby
class User < ActiveRecord::Base
    def self.search(terms, page)
        columns = %w( name login location country )
        tokens = terms.split(/\s+/)
        
        if tokens.empty? 
            conditions = nil
        else
            conditions = tokens.collect do |token|
                columns.collect do |column|
                    "#{column} LIKE '%#{connection.quote(token)}%'"
                end 
            end
            conditions = conditions.flatten.join(" OR ") 
        end

        paginate :conditions => conditions, :page => page 
    end
end

class UsersController < ApplicationController
    def index
        @users = User.search(params[:search], params[:page])
    end 
end
```

#### Solution: Use a proper Full-Text Search Engine
There are many full-text search engines with smooth integrations with rails.
The selection of search engine depends on many factors that are specific to your application and infrastructure. That discussion is beyond the scope of this summary.  However, [here is a convenient list](https://github.com/markets/awesome-ruby#search) of many of the search engine integrations available for ruby.

>Note: Postgres has a built-in full text search engine which may be good enough for you application.  Before adding a new dependency, take a look at the Postgres engine to check if it suits your needs.

The book shows an example on how to integrate with Sphinx using [Thinking Sphinx](https://github.com/pat/thinking-sphinx). I will omit this explanation as it would be obsolete for any other engine that is not Sphinx.

## 1.4 Anti-pattern: Duplicate Code Duplication
### 1.4.1 Problem: 2 Classes behave the _exact_ same way and they repeat the code
Consider this example: a `Car` and a `Bicycle` class have the exact same implementation of all methods that make them _drivable_.

```ruby
class Car << ActiveRecord::Base 
    validates :direction, presence: true 
    validates :speed, presence: true

    def turn(new_direction) self.direction = new_direction end
    def brake self.speed = 0 end
    def accelerate self.speed = speed + 10 end
    # Other, car-related activities... 
end

class Bicycle << ActiveRecord::Base 
    validates :direction, presence: true 
    validates :speed, presence: true

    def turn(new_direction) self.direction = new_direction end
    def brake self.speed = 0 end
    def accelerate self.speed = speed + 10 end
    # Other, bike-related activities... 
end
```
Note that this code repeats the implementation for `turn`, `brake` and `accelerate`; as well as the validations.

#### Solution: Extract shared behavior into a `Drivable` module
We can abstract the concept by thinking that both Cars and Bicycles are _Drivable_.  With this in mind, we can extract all the behavior that makes something `Drivable` into a module.

```ruby
# lib/drivable.rb 
module Drivable
    extend ActiveSupport::Concern
    included do
        validates :direction, presence: true 
        validates :speed, presence: true
    end
    def turn(new_direction) self.direction = new_direction end
    def brake self.speed = 0 end
    def accelerate self.speed = speed + 10 end
end

class Car << ActiveRecord::Base 
    include Drivable
    # Other, car-related activities...
end

class Bicycle << ActiveRecord::Base 
    include Drivable
    # Other, bike-related activities...
end
```
> This could also be achieved by implementing a superclass.  However, most of the times it is preferable to use modules instead of super classes, as modules are more flexible and allow for sharing of multiple types of behavior without most of the problems that come with multiple inheritance. (e.g a Car may be both `Drivable` and `Bookable`). 


### 1.4.2 Problem: 2 Classes have _very similar_ behavior and some of the code is repeated
__Behavior in the 2 classes only changes slightly__

Go back to hour previous example but now imagine we had to add the following constraints:
* `Cars` have a top speed of 100 mph and accelerate at a rate of 10 mph.
* `Bicycles` have a top speed of 20 mph and accelerate at a rate of 1 mph.

The rest of the behavior that makes them `Drivable` stays the same.

In a NON-DRY implementation, the code would look like this:
```ruby
class Car << ActiveRecord::Base 
    validates :direction, presence: true 
    validates :speed, presence: true

    def turn(new_direction) self.direction = new_direction end
    def brake self.speed = 0 end
    def accelerate self.speed = [speed + 10, 100].min end
    # Other, car-related activities... 
end

class Bicycle << ActiveRecord::Base 
    validates :direction, presence: true 
    validates :speed, presence: true

    def turn(new_direction) self.direction = new_direction end
    def brake self.speed = 0 end
    def accelerate self.speed = [speed + 1, 20].min end
    # Other, bike-related activities... 
end
```

#### Solution: Extract shared behavior into a `Drivable` module AND customize particular behavior through the template pattern

The solution is very similar to the previous solution. However, now all `Drivable` classes have the responsibility to define their top speed and acceleration through methods.

Note that in the following code the `Drivable` module enforces all implementers to define their own `#top_speed` and `#acceleration` methods to be able to `#accelerate`.  `Drivable` however, helps a bit by raising informative exceptions in case a developer forgets to implement any of these.

```ruby
# lib/drivable.rb 
module Drivable
    extend ActiveSupport::Concern
    included do
        validates :direction, presence: true 
        validates :speed, presence: true
    end
    def turn(new_direction) self.direction = new_direction end
    def brake self.speed = 0 end
    def accelerate self.speed = [speed + acceleration, top_speed].min end
    
    def top_speed
        raise TemplateError, "The Drivable module requires the " +
                             "included class to define a " + 
                             "top_speed method"
    end

    def acceleration
        raise TemplateError, "The Drivable module requires the " +
                             "included class to define an " + 
                             "acceleration method"
    end
end

class Car << ActiveRecord::Base 
    include Drivable
    # Implementation of template methods
    def top_speed 100 end
    def acceleration 10 end
    # Other, car-related activities...
end

class Bicycle << ActiveRecord::Base 
    include Drivable
    # Implementation of template methods
    def top_speed 20 end
    def acceleration 1 end
    # Other, bike-related activities...
end
```

_Why use methods instead of constants?_

* Methods provide flexibility: for a case of an amphibian vehicle, the top_speed and acceleration may change if the vehicle is currently on water or land.  Methods provide the flexibility for such implementation.
* Easiness of testing: it is more natural to test methods than constants and it is easier to stub methods than to stub constants.
* We can raise helpful error messages through methods.
