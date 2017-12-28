# Rails Anti-patterns
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

#### Solution: Separate the scopes on the models they belong and use scopes
Here are 2 alternatives
```ruby
class User < ActiveRecord::Base 
    has_many :memberships
    def find_recent_active_memberships memberships.find_recently_active end 
end

class Membership < ActiveRecord::Base 
    belongs_to :user
    def self.find_recently_active
        where(:active => true).limit(5).order("last_active_on DESC")
    end 
end
```

```ruby
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


