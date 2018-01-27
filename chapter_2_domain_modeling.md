# Chapter 2 - Domain Modeling
## 2.1 Set of problems: Authorization code is messy and/or over-engineered

Authorization is a hard problem (period). Here is a list of the most common problems that authorization codes have.

### 2.1.1 Problem: The `User` model defines convenience methods that determine the authorization rules
```ruby
class User < ActiveRecord::Base
    # Other roles related code...
    def can_edit_content?
        self.has_roles?(['admin', 'editor', 'associate editor'])
    end
    def can_edit_post?(post) 
        self == post.user ||
        self.has_roles?(['admin', 'editor', 'associate editor']) 
    end
end
```

Problems:
* It is not the `User` responsibility to authorize.
* As the `can_edit_post?` method depicts, many authorization rules involve instances of other models (e.g post). This gets really messy really fast.

#### Solution: See authorization best practices according to your needs.

### 2.1.2 Problem: There is no single authoritative source of what the roles are.

Go back and check the code of the previous example.  All convenience methods repeat the names of the roles (_i.e admin, editor, associate editor_).  This means that if you need to change something, you have to do it in many places AND if you forget a change, it will fail silently.

#### Solution: See authorization best practices according to your needs.
Any of the strategies shown there has a single authoritative source of roles.

### 2.1.3 Problem: Over-engineered authorization code
Quite often, developers program overly complex authorization code that is not really needed for the scope of the application.

#### Guidelines to avoid over-engineered solutions
* Never build beyond the application requirements at the time you are writing the code.
* If you do not have concrete requirements, don't write any code.
* Don't jump to a model prematurely; there are often simple ways such as using Booleans and denormalization to avoid using additional models.
* If there is no user interface for adding, removing or managing data, there is no need for a model. A denormalized column populated by a hash or array of possible values is fine. See example:

```ruby
# On an application where roles have no use interface
class Role < ActiveRecord::Base
    belongs_to :user
    # Denormalized but centralized role names
    TYPES = %w(admin editor writer guest) 
    validates :name, :inclusion => {:in => TYPES}
end
```

### 2.1.4 Overarching Solution: Authorization best practices
* Go with the simplest approach that your requirements permit.
    * Simple is not the same as stupid. If your authorization code is repetitive, is mixed with code of other nature (among others smells), your code is stupid, not simple.
* [Daniel Kehoe's capstone tutorials](https://tutorials.railsapps.org/) have a great series on authorization.  In there he discusses and compares the pros and cons of different complexity level approaches to authorization.
* I have some [google slides](https://docs.google.com/presentation/d/1XOxhIV3LCAlNh4ghn75nkRRWPMb_CnsR3h_DBIqX3FI/edit?usp=sharing) where I explain and compare in good detail some of the typical approaches to authorization (including advanced authorization features).  I'm sorry they are currently in Spanish.
 
## 2.2 Problem: Too many "mini models" to represent things like states, categories, etc

Consider an app where `Articles` have states (e.g published) and categories (e.g faqs).
Such app could have the following _normalized_ code:

 ```ruby
 class Article < ActiveRecord::Base
    belongs_to :state
    belongs_to :category

    validates :state_id, presence: true
    validates :category_id, presence: true
 end

 class State < ActiveRecord::Base
    has_many :articles
    
    # Dynamically defines convenience methods to
    # do things like State.published
    class << self 
        all.each do |state|
            define_method "#{state}" do 
                find_by(name: state)
            end 
        end
    end
 end

 class Category < ActiveRecord::Base
    has_many :articles
    # Equivalent dynamic code for Category.faqs
 end

 # With this code we can do things like
 @article.state = State.published
 @article.state == State.published
 ```
Despite being the _normalized_ approach to the problem, this code has some problems:

* Adding this 2 'little extra' models (`State` and `Category`) bring a ton of code with them: think about code in the models themselves, migrations, specs, factories, etc.
* Very often this 'little' models do not even require a user interface because they are tightly coupled with application logic. Not even admins should be allowed to modify them.
    * Think what will happen if an admin changes the name (or deletes) a state that has behavior tied to it in your code... __Everything will break! __
    * You are better of not even allowing the change in the first place by not storing that in the DB. 

> Rule of thumb: If there is no user interface for adding, removing or managing data, there is no need for a model. A denormalized column populated by a hash or array of possible values is fine.

### Solution 1 (hand-rolled): Denormalize states and categories into text fields
> Included because the meta-programming concepts can be very useful in other contexts. Next solution is more suitable for this case.

With a little help of meta-programming magic, we can completely eliminate the `State` and `Category` models and at the same time define convenient interfaces for handling the `Article's` states and categories.

```ruby
class Article < ActiveRecord::Base
    STATES = %w(draft review published archived)
    CATEGORIES = %w(tips faqs misc)

    validates :state, inclusion: { in:  STATES}
    validates :category, inclusion: { in:  CATEGORIES}
    
    # Dynamically defines instance methods of the type
    # article.draft?
    STATES.each do |state|
        define_method "#{state}?" do
            self.state == state
        end
    end

    # Similar dynamic code for categories...

    class << self
        # Dynamically defines class methods to return the string of the state
        # e.g article.state = Article.draft 
        # Article.draft # => "draft"
        STATES.each do |state|
            define_method "#{state}" do
                state
            end
        end

        # Similar dynamic code for categories...
    end
end
```

### Solution 2 (preferred): Use Rails Enums
Rails has a feature called [Enums](http://api.rubyonrails.org/classes/ActiveRecord/Enum.html) that was made for this exact reason.

This was already covered previously in the summary.

## 2.3 Problem: Too many little models for small non-structured data
Consider a scenario where you want to build the back end for the interface shown next.  Note that if the user selects the "other" option, it must provide a short text in the text box.
<img src="/images/ch2/how_did_you_hear_about_us.png" width="600"/>

We will compare the pros and cons of multiple approaches to model this problem.

### Approach 1: The "Has many and Belongs to Many" normalized solution
```ruby
class User < ActiveRecord::Base 
    has_many :referrals
end

# There is a join table "referral-users" in between

class Referral < ActiveRecord::Base 
    has_and_belongs_to_many :users
end
```

Pros: 

* If the `Referral` categories is something you may want to edit through the admin (without changing the code), then this is the way to go.

Cons:

* If `Referral` categories is not something you want to change through the admin, then this is overkill and introduces too much code and 2 new models for a very small functionality.
* You still need to house a `referral_other` attribute in the User (or the join table) to store the user's text. This is kind of weird.

### Approach 2: "Has Many" slightly de-normalized solution.
```ruby
class User < ActiveRecord::Base 
    has_many :referral_types
end

# Has attributes user_id, value
class Referral < ActiveRecord::Base 
    VALUES = ['Newsletter', 'School', 'Web',
              'Partners/Events', 'Media', 'Other']
    validates :value, inclusion: {in: VALUES}
    belongs_to :user 
end
```

Pros:

* This requires 1 model less than the previous approach (no join table).

Cons:

* We can no longer edit the Referral Categories through the data base. (You need to change the code).
* We still have to keep the `referral_other` attribute in the `User` model.

### Approach 3: Use boolean flags
```ruby
# User has attributes (all booleans, except other):
#   heard_through_newsletter, heard_through_school, heard_through_web,
#   heard_through_partners, ..., heard_through_other (text) 
class User < ActiveRecord::Base  end
```
Pros:

* No extra models at all. Use this ONLY if you need maximum 3 booleans.
* Code is extremely simple (no extra code at all).

Cons:

* If you need more than 3 booleans, this gets really messy.
* Changes in referral types require a migration!

### Approach 4: Store data as a serialized hash in a text column on the `User` model

Serialization is the process of converting an object/hash/array into it's text representation for storage in the database (and the other way around for data retrieval). 

This means that we can turn the ruby hash  `{"Newsletter" => true, "School" => false, "Other" => "Street Flyer"}` to a text (typically YAML of JSON) to store in the DB as text.

Active Record attribute serializers doc all the heavy lifting of serializing and de-serializing when records are saved or retrieved.

For this example the code would be something along the lines of: 
```ruby
class User < ActiveRecord::Base
    HEARD_THROUGH_VALUES = ['Newsletter', 'School', 'Web',
        'Partners/Events', 'Media', 'Other'] 

    # Which attribute to serialize and which serializer to use
    serialize :heard_through, JSON
end
```

The view would look like this:
```html
<%= fields_for :heard_through, (form.object.heard_through||{}) do |heard_through_fields| -%>
    <% User::HEARD_THROUGH_VALUES.each do |heard_through_val| -%> 
        <%= heard_through_fields.check_box "field %>
        <%= heard_through_fields.label :heard_through, heard_through_val %>
    <% end -%> 
<% end -%>
```
> The book authors don't make clear how to handle the "other" text in this case.  Presumably you could store the text directly inside the hash as the value of the "other" key.  However, this will require a little bit of front-end magic.

Pros:

* No extra models.
* Highly flexible structure.

Cons:

* Changes in referral types require changes in code.
* You loose search capabilities in the DB. 
    * Use this if you are certain that you don't require to search by the `heard_through` attribute.
    * You could try to do string search. However, this gets very complicated very fast.   

### Approach 5 (Bonus): Use Postgres' native de-normalized column types (JSONB or Arrays)

Postgres supports array and jsonb type columns natively.  This means that you can store, index and query de-normalized data. Jsonb columns integrate smoothly with Rails as Hashes (no effort needed on our part).

The explanation on how to use these features is out of the scope of this summary. However, here are some useful resources about it:
- https://nandovieira.com/using-postgresql-and-jsonb-with-ruby-on-rails
- http://blog.plataformatec.com.br/2014/07/rails-4-and-postgresql-arrays/

Pros: 

* No extra models.
* Highly flexible structure.
* Data is searchable and indexable.

Cons: 

* Changes in referral types require changes in code.


## 2.4 Bonus Problem: Trying to enforce a normalized structure to open-ended data.

> Authors include this problem inside the "Make use of Rails Serialization" section (mixed inside the previous problem).  However, this problem is different and important enough to deserve it's own section.

It is fairly common in applications to handle data whose structure can't be anticipated.  For example an e-commerce platform where the sellers specify custom specs for the products: 

- Product A has specs: Max-speed: 10 km/h, Available colors: red, green, blue.
- Product B has specs: Size: Medium.

It is a mistake to try to come up with an overly clever normalized solution to support the dynamic nature of this data.

This is a very simple example; however, the solution still holds for applications that need to handle more complicated non-structured data (see book for examples).

### Solution: Embrace the dynamic nature of the data and use serialization, JSONB or Array data types

Take a look at [approaches 4 and 5](#approach-4-store-data-in-as-a-serialized-hash-in-a-text-column-on-the-user) of the previous problem.  You may use these same approaches to solve your problem
