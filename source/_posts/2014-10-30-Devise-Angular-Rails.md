---
layout: post
title: "Wiring Devise into Angular + Rails"
date: 2014-10-30 16:17
comments: true
categories: Rails, Angular, Devise
---

### Walk-through to use Devise with Angular in Rails 4.2.

This is more of a jump-through than a walk-through since I am going to be more jumping around the code than really walking it slowly. The reason for this is that I think it will be more valuable to have a general idea of how the wiring works than an excessively detailed view that may not apply to most cases (my example was very basic).

Include in your Gemfile devise and bundle. We request at least version 4.3.1 because of some troubles with previous versions and Rails 4.2.

```ruby
gem 'devise', '~> 3.4.1'
```

Run the generator and follow the config steps suggested by the gem.

```bash
$ rails g devise:install
```

Make devise respond to json by adding the following to your application.rb:

```ruby
config.to_prepare do
  DeviseController.respond_to :html, :json
end
```

<!-- more -->

Generate the User model with devise:

```bash
$ rails g devise user
```

Don't forget to floss, I mean `rake migrate`. Now, download the following modules from the angular site: [angular-cookies.js, angular-route.js] and place them where you have your angular file (in my case in app/assets/javascripts/). Make sure to add them to the application.js manifest after loading angular:

```js
//= require jquery
//= require jquery_ujs
//= require angular.min
//= require angular-cookies.min
//= require angular-route.min
...
//= require_tree .
```

Now, in my case I use the application layout to call the angular app.

```erb
<!-- application.html.erb -->
<!DOCTYPE html>
<html data-ng-app="WhateverApp">
  <head>...</head>
  <body>
    <%= yield %>
  </body>
</html>
```

Again, in my particular case all I have is a single Rails controller with a single action (index), and a single index view. In that index.html.erb we reference to angular routes:

```erb
<div data-ng-view></div>
```

In app/assets/javascript/angular_app/ we create a new file `routes.js` with the routes we want. In my case I have appended the .erb type so Rails pre-processes the erb tags:

```javascript
(function() {
  var app = angular.module('GemStore');
  app.config(['$routeProvider', function($routeProvider) {
    $routeProvider
      .when('/store_front', {
        controller: 'Resource1Controller',
        templateUrl: '<%= asset_path("angular_app/resource_1/views/store_forefront.html") %>'
      })
      .when('/admin', {
        controller: 'UsersCtrl',
        templateUrl: '<%= asset_path("angular_app/users/views/user_entrance.html") %>'
      })
      .when('/dashboard', {
        controller: 'Resource2Ctrl',
        templateUrl: '<%= asset_path("angular_app/resource_2/views/dashboard.html") %>'
      })
      .otherwise({
        redirectTo: '/store_front'
      });
  }]);
})();
```

The route /admin will take us to the User that Devise created and whose view is:

```javascript
<div data-ng-controller="UsersCtrl">
  <div>
    <div data-ng-show="!isLoggedIn()">
      <button data-ng-click="setForm('signUp')"> Create User</button>
      <button data-ng-click="setForm('signIn')">Sign In</button>
    </div>
    <div data-ng-show="isLoggedIn()">
      <button data-ng-click="signOut()">Sign Out</button>
    </div>
  </div><br><br>

  <form name="newUserForm" data-ng-show="form == 'signUp'">
    <label >Email</label>:
    <input type="text" required data-ng-model="newUser.email"></br>
    <label >Password</label>:
    <input type="password" required data-ng-model="newUser.password"></br>
    <label >Password confirmation</label>:
    <input type="password" required data-ng-model="newUser.password_confirmation"></br>

    <button data-ng-disabled="newUserForm.$invalid" data-ng-click="signUp(newUser)"> Create User</button>
  </form>

  <form name="loginForm" data-ng-show="form == 'signIn'">
    <label>Email</label>
    <input type="text" required data-ng-model="user.email"></br>
    <label>Password</label>
    <input type="password" required data-ng-model="user.password"></br>
    <button data-ng-disabled="loginForm.$invalid" data-ng-click="signIn(user)">Sign In</button>
  </form>
</div>

<button data-ng-click="StorePath()">Store Front</button>
```

The different panels and buttons show and hide depending on if we are logged in or not. I included those functions in my top app.js and under the rootScope so they can be called from anywhere:

```javascript
app.run(['$rootScope','$cookieStore', '$location', function($rootScope, $cookieStore, $location) {
    $rootScope.isLoggedIn = function() {
      return ($cookieStore.get('logged_user') ? true : false);
    };

    $rootScope.logged_user = function() {
      return $cookieStore.get('logged_user').email;
    };
  }]);
```

Finally, the UserCtrl.js includes the way to connect to Rails, query the User model and set the cookies:

```javascript
(function(){
  var app = angular.module('GemStore');

  app.controller('UsersCtrl', [
    '$scope','$http','$cookieStore','$location',function($scope, $http, $cookieStore,$location){

    $scope.user = {};
    $scope.new_user = {};

    $scope.signUp = function(new_user){
      data = { user: new_user };
      $http.post('/users', data)
      .success(function(data){
        $scope.new_user = {};
        $cookieStore.put('logged_user', data);
        $scope.setForm('');
        $location.path('/dashboard');
      })
      .error(function(data, status){
        console.log(data);
        console.log(status);
      });
    };

    $scope.signIn = function(new_user){
      data = { user: new_user };
      $http.post('/users/sign_in', data)
      .success(function(data){
        $scope.user = {};
        $cookieStore.put('logged_user', data);
        $scope.setForm('');
        $location.path('/dashboard');
      })
      .error(function(data,status){
        console.log(data);
        console.log(status);
      });
    };

    $scope.signOut = function() {
      $http({
        method: 'DELETE',
        url: 'users/sign_out'
      })
        .success(function() {
          $cookieStore.remove('logged_user');
          $scope.setForm('');
          $location.path('/store_front');
        })
        .error(function(status) {
          console.log(status);
        })
    };

    $scope.setForm = function(form) {
      $scope.form = form;
    };
  }]);
})()
```

Finally, make sure you are nullifying the sessions as you protect from CSRF attacks.

```ruby
class ApplicationController < ActionController::Base
  respond_to :html, :json

  # Prevent CSRF attacks by raising an exception.
  # For APIs, you may want to use :null_session instead.
  # protect_from_forgery with: :exception
  protect_from_forgery with: :null_session
end
```

Special thanks to Derek Maffett, whose help was invaluable to navigate the angular example.




