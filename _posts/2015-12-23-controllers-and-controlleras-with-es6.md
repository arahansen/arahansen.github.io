---
layout: post
title: Controllers and controllerAs with ES6
categories: angular
tags: angular es6
---

If you've been paying attention to best practices for Angular 1.x recently, you might have noticed a shift towards defaulting to `controllerAs` when defining your controllers to isolate parent and child controllers from each other. This provides a really nice barrier that prevents unintended inheritance and aligns well with the whole "separation of concerns" for which we all strive in our modern web apps.

controllerAs syntax also allows for a more declarative approach to building our views; making abundantly clear where we are binding data to a given view.

## ControllerAs in ES5

In ES5, one popular approach for utilizing controllerAs syntax looks like this:

{% highlight html %}
--- my-view.html ---

<!-- NOTE: using ng-controller is not usually the recommended approach for
 defining controllers and controllerAs you should define your controller in
 your routes, or, if using making use of Angular 1.5's component feature,
 in your component definition (which uses controllerAs by default)
-->
<div ng-controller="myCtrl as ctrl">
  {% raw %}{{ctrl.name}}{% endraw %}
  <button ng-click="ctrl.changeName()">
    Change Name
  </button>
  <br>
  <input type="radio" ng-model="ctrl.color" value='the white'> White
  <br>
  <input type="radio" ng-model="ctrl.color" value='the gray'> Gray    
</div>
{% endhighlight %}

{% highlight javascript %}
--- my-controller.js The Old, ES5 Way---

angular.module('myApp', [])
  .controller('myCtrl', myController);

function myController($scope) {
  var ctrl = this;
  ctrl.name = "gandalf";
  ctrl.color = 'the gray';

  ctrl.changeName = function() {
    function getName() {
      // context of 'this' changes inside getNewName function
      // need ctrl to maintain controller context
      return 'gandalf ' + ctrl.color;
    }
    ctrl.name = getName();
  };
}  
{% endhighlight %}

As you can see, the definition of `this` is in constant flux throughout the controller. We use `var ctrl = this` to combat this and keep the definition of `this` constant.

## The ES6 Way

With the advent of ES6, we gain a lot of really great features. Among those changes are "fat arrow" functions.

Fat arrow functions give us the ability to lexically bind to the surrounding function scope. We can use this to our advantage by building our controller functions using fat arrow syntax to maintain a consistent reference to `this`.

So, if we were to implement our controller using ES6, we would want to construct it like this:

{% highlight javascript %}
--- my-controller.js The New, ES6 Way---

angular.module('myApp', [])
  .controller('myCtrl', myController);

function myController($scope) {
  this.name = "gandalf";
  this.color = 'the gray';

  this.changeName = () => {
    // context remains the same as the parent closure
    // when using fat arrow functions
    var getName = () => {
      return 'gandalf ' + this.color;
    }
    this.name = getName();
  };
}  
{% endhighlight %}

Overall, this ES6 fat arrow approach brings us ever closer to a cleaner, more concise controller that doesn't depend on extraneous variables like `var ctrl = this` to force our JavaScript into doing what we want. ES6 now natively supports what amounts to a more natural flow of the a controller definition.

To see the examples from this post in action, [check out the jsfiddle here](http://jsfiddle.net/77f5dfpm/2/).
