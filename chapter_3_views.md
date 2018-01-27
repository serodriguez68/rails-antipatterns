# Chapter 3 - Views
# 3.1 General Problem: Not using the available View Methods that come with Rails (and hand-rolling the solutions)

Rails provides a solution to many of the common patterns that are used in views.  

There are many helper methods given by Rails to cover here. However, the best practice is the same: __First look for a Rails functionality that addresses your problem before attempting to hand-roll your solution.__

Some of the most popular helpers are shown next.

### 3.1.1 Problem: Hand-rolled forms
There are many ways of getting this wrong:

* Using direct HTML
* Using the `form_tag` helper when a `form_for` helper could be used.
    * The use of  `form_tags` may be a symptom that there is something wrong with your domain modelling.
* Using deprecated or redundatn `form_for`  syntax.

#### Solution: Use the standard `form_for` helper with a `@resource`
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

### 3.1.2 Problem: Hand-rolled iteration through `@collections`
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

#### Solution: Use Rails convention for iterating `@collections` in views

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
