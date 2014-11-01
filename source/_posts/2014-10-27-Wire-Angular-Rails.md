---
layout: post
title: "Wiring Angular into Rails"
date: 2014-10-27 20:01
comments: true
categories: Rails, Angular
---

Given an angular site of static pages, without persistent data, just the bells and whistles that Angular provides for the front end, how do we wire it so it is served from inside a Rails application.

The first step is to generate a new rails app (I used 4.2), for which we'll create a controller that will coordinate the rendering of the angular app. We'll start with a single action that points to a show template, where all the action will take place (it would substitute the index.html page of the Angular app). We call our controller StoreController.

In app/assets/javascripts/application.js we are going to remove turbolinks, because it is repetitive functionality now that we'll be relying on Angular and it tends to screw things up. Actually we could remove it from the Gemfile too.

```
# app/assets/javascripts/application.js
//= require jquery
//= require jquery_ujs
//= require angular.min
//= require launch_store
//= require_tree .
```

We also add a requirement for a custom js file that we'll create, app/assets/javascripts/launch_store.js, and where we declare our Angular app as gemStore.

```
# app/assets/javascripts/launch_store.js
//= require_self
//= require_tree ./angular_store_app
(function(){
  var app = angular.module("GemStore", ["StoreDirectives"]);
})();
```
<!--more-->
In this newly loaded file we reference again another folder tree, 'angular_store_app', where we'll store all the Angular functionality by resources. In our case we are using the resulting app from CodeSchool's course "[Shaping up with Angular](http://campus.codeschool.com/courses/shaping-up-with-angular-js/intro)", a store-like app for displaying gems and jewels. Therefore we'll create a subfolder called 'gems' and inside another three subfolders: controllers, directives, and views.

{% img center /images/Oct14/folders.png 300 %}

Our convention is going to be to name directives in camelCase, and everything else in PascalCase.

Now all it is left to do is to add the controllers and directives to its respectives files:

```javascript
// gems_controller.js
(function(){
  var app = angular.module('GemStore');
  app.controller('StoreController', ['$http', '$scope', function($http, $scope) {
    ...
  ]);
})();

(function(){
  var app = angular.module('GemStore');
  app.controller('ReviewController', ['$scope', function($scope) {
  ...
  });
})();

// gems_directives.js
(function() {
  var app = angular.module("StoreDirectives", []);
  app.directive("productDescription", function() {
    return {
      restrict: "A",
      templateUrl: "<%= asset_path('angular_store_app/gems/views/description.html')%>"
    };
  });
...
```

For the rails views we need to, first of all, make sure our routes are well defined and pointing to 'store#show'.

Then in the application.html.erb we reference the angular app (check that we have taken out the turbolinks):

```html
<!DOCTYPE html>
<html data-ng-app="GemStore">
<head>
  <title>Angurails</title>
  <%= stylesheet_link_tag 'application', media: 'all' %>
  <%= javascript_include_tag 'application' %>
  <%= csrf_meta_tags %>
</head>
<body>

<%= yield %>

</body>
</html>
```

And in the show.html.page of the store controller we can start coding our own angular references and expressions:

```html
<div data-ng-controller="StoreController">
  <header>
    <h2 class="text-center">Soto's Magnificent Emporium</h2>
    <h3 class="text-center"> an sinangular store </h3>
  </header>

  <div class="list-group">
    <div class="list-group-item" data-ng-repeat="product in products">
      <h3>{{product.name}} <em class="pull-right">{{product.price | currency}}</em></h3>
      <div data-product-gallery></div>
      <div data-product-tabs></div>
    </div>
  </div>
</div>
```

One last details is to include in the rails public folder our angular html snippets referenced by directives, and make sure that the url in the gems_directives.js are correct.

```html
<!-- app/assets/javascript/angular_store_app/gems/views/description.html -->
<div data-ng-show="tab.isSet(1)">
  <h4>Description</h4>
  <blockquote>{{product.description}}</blockquote>
</div>
```

That's it. Run it and check it works.

One final note of advice, in my case using CoffeeScript gave me quite a bit of trouble, and although I still have to investigate the exact causes of the it, I recommend to use it with caution.
