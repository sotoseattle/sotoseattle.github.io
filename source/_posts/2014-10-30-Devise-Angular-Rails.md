---
layout: post
title: "Adding Devise to Angular inside Rails"
date: 2014-10-29 16:17
comments: true
categories: Rails, Angular, Devise
published: false
---

### Walkthrough to use Devise with Angular in Rails 4.2.

Include in your Gemfile devise and bundle. We request at least version 4.3.1 because of some troubles with lower versions and Rails 4.2.

```ruby
gem 'devise', '~> 3.4.1'
```

Run the generator

```bash
$ rails g devise:install
```

Make devise respond to json by adding the following to your application.rb:

```ruby
config.to_prepare do
  DeviseController.respond_to :html, :json
end
```

Generate the User model with devise:

```bash
$ rails g devise user
```

Download the following modules from the angular site: [angular-cookies.js, angular-route.js] and place them where you have your angular file (in my case in app/assets/javascripts/.

Make sure to add them to the application.js manifest after loading angular:

```js
//= require jquery
//= require jquery_ujs
//= require angular.min
//= require angular-cookies.min
//= require angular-route.min
...
//= require_tree .
```

Create a folder from where to serve your angular views. I used app/public/templates/.

Now, in the application layout we have the call to the angular app:

```erb
<!DOCTYPE html>
<html data-ng-app="WhateverApp">
  <head>...</head>
  <body>
    <%= yield %>
  </body>
</html>
```

...and in the index view page of our admin controller we reference the angular template:

```erb
<div data-ng-view='public/templates/whatever_view'>
  <!-- everything from here -->
</div>
```

...extracting all the index contents and placing them now in the public/templates/admin_view.html

```html
<!-- everything from here goes now here -->
```







