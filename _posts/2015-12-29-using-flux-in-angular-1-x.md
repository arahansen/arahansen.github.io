---
layout: post
title: Using Flux in Angular 1.x
categories: flux
tags: angular flux alt es6
---

Most flux implementations worth using like [redux](https://github.com/rackt/redux) and [alt](https://github.com/goatslacker/alt) make no assumptions about the application that is consuming data updates and triggering actions; flux libraries can be used to handle data regardless of the makeup of the rest of an application's stack.

However, in the vast majority of examples and guides for using a given flux library, authors gravitate towards explaining the library's API using React as the "application stack" that is utilizing the library. Even Facebook's own [flux documentation](https://github.com/facebook/flux) makes reference to sending data specifically to React views.

This approach to documentation can really obscure the fact that flux is a pattern that can be utilized in any web application. In fact, it actually lends itself quite nicely to an Angular context and can simplify shared data and state within an Angular application.

## Sharing Data in Angular

Typically, when sharing data between controllers in Angular, we might write a service that stores data. That service can then be injected into each controller that is concerned about said data.

For example:

{% highlight javascript %}
// --- main.js --- //
angular.module('myApp', [])
.controller('firstController', function(sharedData) {
  this.name = sharedData.name;
})

.controller('secondController', function(sharedData) {
  this.name = sharedData.name;
})

.service('sharedData', function() {
  this.name = 'luke skywalker';
});
{% endhighlight %}

In this simplified example, `firstController` and `secondController` both have access to the datapoint 'name' within the `sharedData` service. While the service could be expanded upon using getters and setters, or even using `$broadcast` and `$emit` to maintain control over how data is updated, this approach is not easily maintainable for larger applications.

What we would find is that state mutation is difficult to track using the above approach. `firstController` and `secondController` are able to update the data within `sharedData` whenever and however they feel like it. In more complex controllers, state might be mutated in far more subtle ways, reducing developer understanding of the application as a whole.

## Implementing Flux in Angular

We can simplify the mutation of data and state by implementing a flux library to handle our data (in today's example, [alt.js](http://alt.js.org/)) within the context of our Angular applications. The result is a set of Angular controllers that have zero impact on the data they are concerned about, and instead data is mutated in a central location.

To picture this, we are going to build an application that uses a couple controllers to toggle some arbitrary messages as read or unread. To see the finished product, check out [this jsbin](https://jsbin.com/laceci/edit).

### Creating the View

Our view is rather simple and is the least interesting part of the app. Right off the bat, it won't do anything (or render much) until we implement controllers and factories. The html is provided up front primarily as reference to the rest of the application implementation.

{% highlight html %}
<!-- index.html -->
<div ng-app='myApp'>
  <h3>First Controller</h3>
  <div ng-controller='firstCtrl as first'>
    read | message
    <div ng-repeat="msg in first.messages">
      <input type="checkbox"
        ng-model="msg.read"
        ng-change="first.toggleRead(msg)"> |
        {% raw %}{{msg.message}}{% endraw %}
    </div>
  </div>
  <hr>
  <h3>Second Controller</h3>
  <div ng-controller='secondCtrl as second'>
    Read Messages: {% raw %}{{second.readCount}}{% endraw %}
    <br>
    <button
      ng-click="second.setAllRead(true)">
      Mark All Read
    </button>
    <button
      ng-click="second.setAllRead(false)">
      Mark All Unread
    </button>
  </div>
</div>
{% endhighlight %}

### Creating an Instance of Alt.js

The first "real" step is to generate an instance of Alt as a singleton that can be used by all of our data stores and action generators. This way, alt is tracking all of our stores and actions within our application using a single, unified instance.

{% highlight javascript %}
// --- alt-factory.js --- //
app.factory('alt', function() {
  return new Alt();
});
{% endhighlight %}

### Creating an Actions Factory

With an alt instance built, we can inject it into our action generator.

{% highlight javascript %}
// --- message-actions.js --- //
app.factory('messageActions', function(alt) {
  return alt.generateActions(
    'updateMessage',
    'setAllRead'
  );
});
{% endhighlight %}

The actions we generate here will match the names of functions within the data store we build next. We will inject these actions into our controllers to trigger an update to the corresponding data store.

When used within our controllers, we will have access to `messageActions.updateMessage` and `messageActions.setAllRead` actions that will trigger an update to the messageStore.

With our actions set up, we are ready to wire the actions up to our data store.

### Initializing the Store Factory

{% highlight javascript %}
// --- message-store.js --- //
app.factory('messageStore', function(alt, messageActions) {
  class MessageStore {
    constructor() {
      // actions are bound to corresponding store functions
      this.bindActions(messageActions);

      // initialize mock data
      this.messages = [
      {
        read: false,
        message: 'one',
        id: 0
      },
      {
        read: true,
        message: 'two',
        id: 1
      },
      {
        read: true,
        message: 'three',
        id: 2
      }];
    }

    // function name matches name of action generated in `message-actions.js`
    updateMessage(message) {
      const index = this.findIndexById(message);
      if (typeof index === 'undefined') throw 'Could not find message';
      else {
        this.messages[index] = message;
      }
    }

    setAllRead(status) {
      this.messages.forEach((elem) => elem.read = status);
    }

    // finds the index of the provided message within 'this.messsages'
    findIndexById(msg) {
      return this.messages.reduce(
        (prev, curr, index) => curr.id === msg.id ?
          index : prev, undefined);
    }
  }

  // this is the same instance of alt
  // (ie, singleton) that was injected into the action factory
  return alt.createStore(MessageStore, 'MessageStore');
});
{% endhighlight %}

The constructor method initializes our data (which could also come from an external source or `$http` request), and binds our message actions (generated in `message-actions.js`) to the store. This essentially wires up the actions to the store so that when an action is triggered in our controller, the proper store knows it should update.

Also note the `updateMessage` and `setAllRead` functions are named such because they match up exactly with the names of the actions we generated.

With our store and actions written, we are ready to make use of our alt instance within our controllers.

### Triggering Actions From Controllers

{% highlight javascript %}
// --- first-controller.js --- //
app.controller('firstCtrl', function(messageStore, messageActions,
                                   $scope) {
  this.toggleRead = (msg) =>         
    messageActions.updateMessage(msg);

  const onChange = (state) => this.messages = state.messages;

  init();
  function init() {
    onChange(messageStore.getState());  
    messageStore.listen(onChange);
    $scope.$on('$destroy', () => messageStore.unlisten(onChange));
  }
});
{% endhighlight %}

`first-controller.js` updates the 'read' status of individual messages. The difference here from typical Angular controllers is that the controller itself does none of the data mutation. It merely lets the store know (via actions) that data needs to be updated in a particular manner (designated by the specific action called).

In `second-controller`, `this.setAllRead` is bound to the `setAllRead` action within the `messageActions` factory. So, when the user clicks 'Mark All Read', or 'Mark All Unread' all the controller is doing is sending a boolean out to `messageStore` via the `setAllRead` action. What the store does with that boolean is completely independent of what the controller knows about.

{% highlight javascript %}
// --- second-controller.js --- //
app.controller('secondCtrl',
  function(messageActions, messageStore, $scope) {

  this.setAllRead = messageActions.setAllRead;

  // returns the total number of read messages
  const onChange = (state) => {
    this.readCount = state.messages.reduce(
      (prev, curr) => curr.read ? ++prev : prev, 0
    );
  };

  init();
  function init() {
    onChange(messageStore.getState());  
    messageStore.listen(onChange);
    $scope.$on('$destroy', () => messageStore.unlisten(onChange));
  }
});
{% endhighlight %}

We also subscribe to store updates within each controller. When the store is updated, each controller is informed and we re-compute the related view models.

### Flux With Angular In Action

When running the [jsbin example](https://jsbin.com/laceci/edit), we see both controllers are updating their views as actions are being triggered by a single controller. This is the power of flux in action! Neither controller is mutating data at any point in time. They are simply listening for changes from the store, and 'gluing' new updates to their view.

## Flux With Angular Works!

Flux fits in really well with the architecture Angular provides. Controllers are supposed to be the "glue" between your model layer and your views. By not mutating data at all within the controller, and instead triggering an action that will signal the store to mutate data, we abstract mutations away from the controller and into the store-factory.

As your application grows, the flux approach to data flow is far more maintainable than the two-way data binding via data services we typically see in Angular applications.
