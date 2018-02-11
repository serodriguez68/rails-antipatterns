# Chapter 3 - Views

## 3.1 Anti-Pattern: PHPitis

### 3.1.1 General Problem: Not using the available View Methods that come with Rails (and hand-rolling the solutions)

Rails provides a solution to many of the common patterns that are used in views.  

There are many helper methods given by Rails to cover here. However, the best practice is the same: __First look for a Rails functionality that addresses your problem before attempting to hand-roll your solution.__

Some of the most popular helpers are shown next.

#### 3.1.1.1 Problem: Hand-rolled forms
There are many ways of getting this wrong:

* Using direct HTML
* Using the `form_tag` helper when a `form_for` helper could be used.
    * The use of  `form_tags` may be a symptom that there is something wrong with your domain modeling.
* Using deprecated or redundant `form_for`  syntax.

##### Solution: Use the standard `form_for` helper with a `@resource`
Rails is smart when dealing with forms.

```erb
<%= form_for @user do |form| %>
```

If `@user` is new user (that is not yet saved), Rails will render the following html.  Note that the http method, class and id have been derived automatically by Rails.
```erb
<form action="/users" method="POST" class="new_user" id="new_user">
```

If `@user` is an existing user (i.e we are editing the user), Rails will render the following html. Note that the http method `put` is now included in the form.
```erb
<form action="/users/5" method="post" class="edit_user" id="edit_user_5">
    <input name="_method" type="hidden" value="put" />
</form>
```

#### 3.1.1.2 Problem: Hand-rolled iteration through `@collections`
There are many ways of getting this wrong:

* Iterating the collection directly in the template. (see code example)
* Using deprecated syntax.

```erb
<!-- posts/index.html.erb --> 
<% @posts.each do |post| %>
    <h2><%= post.title %></h2>
    <%= format_content post.body %> 
    <p><%= link_to 'Email author', mail_to(post.user.email) %> </p>
<% end %>
```

##### Solution: Use Rails convention for iterating `@collections` in views

```erb
<!-- posts/index.html.erb --> 
<!-- By convention this iterates each post in @posts and calls the partial _post.erb -->
<%= render @posts %>

<!-- posts/_post.html.erb --> 
<h2><%= post.title %></h2>
<%= format_content post.body %> 
<p> <%= link_to 'Email author', mail_to(post.user.email) %> </p>
```

Now the partial `_post.html.erb` is a centralized and authoritative source of how a post should be rendered.

### 3.1.2 Problem: Lack of abstraction of repeated code in the views

Imagine we have to code 2 views. 

* View "A" has a navigation with links to the home page, the FAQs and a series of unique links relevant for view "A".
* View "B" has a navigation with links to the home page, the FAQs and a series of unique links relevant for view "B".

A developer comes up with the following solution:

```erb
<!-- In view A file -->
<ul class="nav">
    <li><%= link_to "Home", root_url %></li>
    <li><%= link_to "Faqs", faqs_url %></li> 
    <li><%= link_to "Relevant links for view A", view_a_url %></li> 
</ul>
<p>The rest of the stuff for view A </p>

<!-- In view B file -->
<ul class="nav">
    <li><%= link_to "Home", root_url %></li>
    <li><%= link_to "Faqs", maps_url %></li> 
    <li><%= link_to "Relevant links for view B", view_B_url %></li> 
</ul>
<p>The rest of the stuff for view B </p>
```
This code has the following problems:

* The code for "Home" and "Faqs" is repeated in both views (and possibly in all views).
* Code for thinks like navs, sidebars and footer should probably live in the `layouts/application.html.erb`. Such file typically includes just the empty shells or shared elements for the general elements. Customization of such  elements can be done inside each view as needed (see solution).

#### Solution: Use `content_for` to insert content into named sections of your layout from your view.

```erb
<!-- Inside the layouts/application.html.erb file -->
<html>
    <body>
        <ul class="nav">
            <li><%= link_to "Home", root_url %></li>
            <li><%= link_to "Maps", maps_url %></li>
            <%= content_for :nav %>
        </ul>

        <div class="main">
            <%= yield %>
        </div>

    </body>
</html>

<!-- In view A file -->
<% content_for :nav do %>
    <li><%= link_to "Relevant links for view A", view_a_url %></li> 
<% end %>
<p>The rest of the stuff for view A </p>

<!-- In view B file -->
<% content_for :nav do %>
    <li><%= link_to "Relevant links for view B", view_b_url %></li> 
<% end %>
<p>The rest of the stuff for view B </p>
```

This is much better! `content_for` allows us to "push markup up" from the view to the layout. Notice that `content_for` pushes data to the __named sections__ in the layout.  Whatever is not inside a `content_for` block, will be passed  to the `yield` call in the layout.  Let's clarify that with an example:

```erb
<!-- This goes in the :nav section of the layout -->
<% content_for :nav do %>
    <li><%= link_to "Relevant links for view A", view_a_url %></li> 
<% end %>

<!-- This is goes in the :sidebar section of the layout -->
<% content_for :sidebar do %>
    The current time is:
    <%= Time.now %>
<% end %>

<!-- This is goes to the yield block in the layout-->
<p>The rest of the stuff for view A </p>
```

##### Other common tricks with `content_for

_Define the `<body>` class for the view (with a default class)_
```erb
<!-- Inside the layouts/application.html.erb file -->
<body class="<%= content_for :body_class || 'default-body-class' %>">

<!-- Inside the faqs view -->
<% content_for :body_class, "faqs" %>
```

_Define the head's `<title>` class for the view (with a default value)_
```erb
<!-- Inside the layouts/application.html.erb file -->
<head>
    <title>Acme Widgets | <%= content_for :title || "Home" %></title>
</head>

<!-- Inside the faqs view -->
<% content_for :title, "FAQs" %>
```

_Conditionally rendering a sidebar in the layout if there is content for it_
```erb
<!-- Inside the layouts/application.html.erb file -->
<% if content_for?(:sidebar) %> 
    <div class="sidebar">
        <%= content_for :sidebar %>
    </div>
<% end %>

```

### 3.1.3 Problem: Having logic in views or view helpers that belongs to the model

Consider the following example:
```erb
<% if current_user && (current_user == @post.user ||
    @post.editors.include?(current_user)) && @post.editable? &&
    @post.user.active? %>
    <%= link_to 'Edit this post', edit_post_url(@post) %>
<% end %>
```
This code is determining whether to display a link to edit a post based on a series of conditions that define if a post is editable.

The problem here is that the logic that determines if a post is editable should belong to the `Post` model itself.

#### Solution: Identify logic that belongs to the models and move it there

The solution is simple, add a method `post_editable_by?` to the `Post` model and modify the view to use that method.

```erb
<% if @post.editable_by?(current_user) %>
    <%= link_to 'Edit this post', edit_post_url(@post) %>
<% end %>
```

Some guidelines to identify where logic belongs to:

* If the method will be used in both views and controllers, then it belongs to the model.
* If the method will only be used in the views, then it should be a view helper.

> Note from the summarizer: View helpers are controversial topic in the Rails world. If your application is small, then you can perfectly use the "Rails Way" helpers without any problem.  However, if your application is large, view helpers can become a mess.  Take a look at _Decorators and Presenters_ instead.  Here are some resources about them:

* [Rails Cast for it](http://railscasts.com/episodes/287-presenters-from-scratch?autoplay=true).
* [GoRails Decorators and Draper](https://gorails.com/series/design-patterns)
* [Draper Gem](https://github.com/drapergem/draper)

### 3.1.4 Problem: Still having complex logic in views after solving the 2 previous problems.
At this point we are already using the appropriate view methods given by Rails and we have already made sure that every logic that belongs to a model, has been moved accordingly.  However, we may still find ourselves having  `if` statements in views that degrade readability of the code.  For example:

```erb
<div class="feed"> 
    <% if @project %>
        <%= link_to "Subscribe to #{@project.name} alerts.", project_alerts_url(@project, :format => :rss), :class => "feed_link" %>
    <% else %>
        <%= link_to "Subscribe to these alerts.",
            alerts_url(format => :rss), :class => "feed_link" %>
    <% end %> 
</div>
<!-- More code related with the view -->
```

In this example, if @project exists, then we render a "Subscribe to #{this project} alerts" button.  Else, we render another button.

The problem here is that we are having the view decide which link to render, despite that the ultimate goal is to render _a_ link.

#### Solution: Move that bit to a custom helper (and/or a view partial)
> As I mentioned earlier, view helpers are a controversial topic. See previous comment for more information.

We can move that bit of code to a custom helper. For example, let's say that this view is part of the `ProjectController` and hence it makes sense to move this inside the `ProjectHelper`. 

You DON'T have to keep Rail's default naming convention for view helpers.  View helpers are globally accessible so naming doesn't matter.  Make sure you use a naming system that helps you keep code organized and DON'T clutter the `ApplicationHelper` with everything.

>The fact that helpers are globally accessible is one of the main reasons why view helpers are controversial.

```ruby
# app/helpers/a_place_where_this_makes_sense_helper.rb
def rss_link(project = nil) 
    content_tag :div, :class => "feed" do
        link_to "Subscribe to these #{project.name if project} alerts.", alerts_rss_url(project),
    end 
end

def alerts_rss_url(project = nil) 
    if project
        project_alerts_url(project, :format => :rss) 
    else
        alerts_url(:rss) 
    end
end
```

Note that the solution splits nicely the _view rendering_ concern from the _finding the appropriate link_ concern into 2 different methods.  Also note that we are leveraging the `content_tag` method to render HTML inside one helper.  This is fine  if your helper includes only a few HTML tags. However, if you will be using lots of markup, use a _view partial_ instead.

> You could argue that the _view rendering_ concern should live inside a _view partial_ from the beginning.  You could be right, it is your call to decide when to promote this from a helper to a partial.

## 3.2 Anti-Pattern: Markup Mayhem
### 3.2.1 Problem: Not using semantic markup
Semantic markup means:
* Every element in the page means something and is labeled with a `class`
or `id` to identify it.
* The right tags should be used for the right content.
* Styling should be done at the CSS level (not directly).

Semantic markup meter: Remove all content from your markup and leave just the HTML tags (with classes and ids).  If your empty tags don't make sense, then your markup is not semantic enough (see example).

```html
<!-- Example of non semantic makrup -->
<div> 
    <div>
        <span style="font-size: 2em;"> 
            I love kittens!
        </span> 
    </div>
    <div>
        I love kittens because they're <span style="font-style: italic">
        soft and fluffy! </span>
    </div>
</div>

<!-- That same markup without the content -->
<div> 
    <div>
        <span style="font-size: 2em;"> 
        </span> 
    </div>
    <div>
        <span style="font-style: italic"></span>
    </div>
</div>
```

Problems with this markup:

* In-line styling.
* Wrongs tags used for content (e.g not using h2 or em tags).
* The markup does not make sense when content is removed.

#### Solution: Use semantic markup
Here is a semantic version of the previous code.

```html
<!-- Example of semantic markup -->
<div id="posts">
    <div id="post_1" class="post">
        <h2>I love kittens!</h2>
        <div class="body">
            I love kittens because they're <em>soft and fluffy!</em>
        </div>
    </div> 
</div>

<!-- That same markup without the content -->
<div id="posts">
    <div id="post_1" class="post">
        <h2></h2>
        <div class="body">
            <em></em>
        </div>
    </div> 
</div>
```

From the stripped down version of the markup we can clearly see that this code has something to do with `posts` and that each `post` has a `heading` and a `body`.

### 3.2.2 Problem: Resorting to spaghetti erb to comply with semantic markup

Your are committed to semantic but you resort to ugly `erb` code to be able to comply with it, ultimately affecting your code readability. 

```erb
<div class="post" id="post_<%= @post.id %>"> 
    <h2 class="title">Title</h2>
    <div class="body">Lorem ipsum dolor sit amet, consectetur... </div>

    <ol class="comments">
        <% @post.comments.each do |comment| %>
            <li class="comment" id="comment_<%= comment.id %>"> 
                <%= comment.body %>
            </li> 
        <% end %>
    </ol> 
</div>
```

The previous code is perfectly semantic, but it has the following problems:

* In-line erb (like `id="post_<%= @post.id %>`) is difficult to follow.
* The labeling of sections according to the resource class is done manually.
    * For example, `class="post"` has been written down manually.   

#### Solution: Use Rails' built in view helpers for semantic markup

Rails provides 2 very handy view helpers: `div_for` and the more general `content_tag_for`. The re-factored version of the previous code shows how to use them.

```erb
<%= div_for @post do %>
    <h2 class="title">Title</h2>
    <div class="body">Lorem ipsum dolor sit amet, consectetur... </div>
    
    <ol class="comments">
        <%= content_tag_for(:li, @post.comments) do |comment| %>
            <%= comment.body %>
        <% end %>
    </ol> 
<% end %>
```

`content_tag_for` is handling both the labeling of the `li` element and the iteration. [Here is the documentation](https://apidock.com/rails/ActionView/Helpers/RecordTagHelper/content_tag_for) for the method.

>__Important Note:__ This re-factored code exhibits an anti-pattern that we covered previously: [Hand-rolled iteration of comments](#3112-problem-hand-rolled-iteration-through-collections), when we could use the more convention friendly `render comments` instead.

Note that standard rails helpers like `form_for` also automatically generate semantic markup for you (for free).


### 3.2.3 Problem: `erb`sucks >  Solution: use [HAMLS](https://github.com/haml/haml) or [Slim](https://github.com/slim-template/slim)

### 3.2.4 Problem: `CSS`sucks >  Solution: use [Sass](http://sass-lang.com/)

