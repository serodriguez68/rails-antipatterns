# Chapter 3 - Views
# 3.1 General Problem: Not using the available Rails View Methods (and hand-rolling the solutions)

Rails provides a solution to many of the common patterns that are used in views.  

There are many helper methods given by Rails to cover here. However, the best practice is the same: __First look for a Rails functionality that addresses your problem before attempting to hand-roll your solution.__

Some of the most popular helpers are shown next.

### 3.1.1 Problem: Hand-rolled forms
There are many ways of getting this wrong a form: 

* Using direct HTML
* Using the `form_tag` helper when a `form_for` helper could be used.
    * The use of  `form_tags` may be a symptom that there is something wrong with your domain modelling.
* Using deprecated or redundatn `form_for`  syntax.

#### Solution: Use the standard `form_for` helper with a `@resource`
Rails is smart when dealing with forms.

```html
<%= form_for @user do |form| %>
```

If `@user` is new user (that is not yet saved), Rails will render the following html.  Note that the http method, class and id have been derived automatically by Rails.
```html
<form action="/users" method="POST" class="new_user" id="new_user">
```

If `@user` is an existing user (i.e we are editing the user), Rails will render the following html. Note that the http method `put` is now included in the form.
```html
<form action="/users/5" method="post" class="edit_user" id="edit_user_5">
    <input name="_method" type="hidden" value="put" />
</form>
```

