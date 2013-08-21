---
layout: post
title: "Searching for a JS MV* solution"
tags: ["javascript", "frontend"]
---

I'm finally getting down to implementing a small-to-medium size project that's been rolling around in my head for awhile now, and it's going to invole some wholesome frontend work -- HTML, CSS, and Javascript. Now I've done all that before, but one of the things I wanted to do differently this time around was have a better approach to model-view separation. HTML doesn't naturally promote this separation, leading me to perform jQuery selector gymnastics in the past in order to deduce things about my model. Instead of simply calling `length` on an array in Javascript, I must count the number of `li` children of a `ul` or `ol` element.

Fuck that noise. MVC has been around forever, and it's used all over in mobile programming and server-side frontend programming. The app I want to build is simple enough that I only really need one-way data binding, meaning I may update the model directly and the views should update in turn. I don't even need two-way data binding, meaning the view accepts input (more specifically, the `input` elements on the page do) that in turn updates the model. Surely something simple out there fulfills my needs.

Searching around for a solution first led me to [Backbone.js](http://documentcloud.github.com/backbone/), which I've tried to pick up many times before and have never gotten anywhere with. I think my problem with Backbone is the way its documentation is organized -- it's just a laundry list of modules and methods, as opposed to having any sort of narrative, explaining each problem and its solution in turn. (The documentation format works well for [Underscore.js](http://documentcloud.github.com/underscore/), however, since it is just a collection of utility methods.)

More searching led me to [this blog post](http://codebrief.com/2012/01/the-top-10-javascript-mvc-frameworks-reviewed/) and a handful of Stack Overflow posts on the subject. One of those posts then led me to [another blog post](http://blog.stevensanderson.com/2012/08/01/rich-javascript-applications-the-seven-frameworks-throne-of-js-2012/), which convinced me that I should be looking at Backbone.js, [Ember](http://emberjs.com/), [Knockout](http://knockoutjs.com/), and [AngularJS](http://angularjs.org/). Now I'd already given up on Backbone. I decided to take a look at Knockout, because one Stack Overflow post in particular called out its simplicity if you only need two-way data bindings.

In addition to looking at Knockout, I also wanted to look at either Ember or AngularJS. Both are "opinionated" and include everything you need to build an ambitious web application, which mine was not. The primary author of Ember is Yehuda Katz, who is a member of the Ruby on Rails core team, and so Ember works especially well with a RESTful, Ruby backend, while mine will not be. Several others praised AngularJS uncaring approach to your backend is like, and also its use of dependency injection to promote testability, and so I decided to also give it a look.

In the end I decided to go with KnockoutJS because it is small, understandable, and provides little more than what I need. While AngularJS provides much more functionality, and is a little overwhelming at times, it definitely left a good impression on me. I could hardly believe that the authors of AngularJS worked for the [same company](http://www.google.com) that is trying to convince everyone that Dart is a good idea.

Below I've included the notes I took while researching both frameworks. They are probably useless without any context, but may prove helpful while reading the documentation. Enjoy.

## KnockoutJS

Built around three features:

* Observables and dependency tracking
* Declarative bindings
* Templating

### Creating view models with observables

MVVM (Model-View-View Model) architecture:

* Model is the data on the server-side, totally independent of any concept of a view.
* View Model is that data on the client-side, and has methods for manipulating that data which the view can call.
* View is the HTML.

A view model without any observables:
{% highlight javascript %}
var myViewModel = {
    personName: 'Bob',
    personAge: 123
};
{% endhighlight %}

A view model with observables:
{% highlight javascript %}
var myViewModel = {
    personName: ko.observable('Bob'),
    personAge: ko.observable(123)
};
{% endhighlight %}

To get:
{% highlight javascript %}
myViewModel.personName()
myViewModel.personAge()
{% endhighlight %}

To set:
{% highlight javascript %}
myViewModel.personName('Mary')
myViewModel.personAge(50)
{% endhighlight %}
 
To populate HTML with observables, use the `data-bind` attribute.

### Using computed observables

Observables can have computed, or derived, values:
{% highlight javascript %}
function AppViewModel() {
    this.firstName = ko.observable('Bob');
    this.lastName = ko.observable('Smith');
    this.fullName = ko.computed(function() {
        return this.firstName() + " " + this.lastName();
    }, this);
}
{% endhighlight %}

Note that `this` is passed in as a last parameter; this binds the `AppViewModel` to `this` in the provided `function`. A common idiom is to use `self`:
{% highlight javascript %}
function AppViewModel() {
    var self = this;
    this.firstName = ko.observable('Bob');
    this.lastName = ko.observable('Smith');
    this.fullName = ko.computed(function() {
        return self.firstName() + " " + self.lastName();
    });
}
{% endhighlight %}

### Working with observable arrays

An `observableArray` tracks which objects are in the array, not the state of those objects. Simply putting an object into an `observableArray` doesn’t make all of that object’s properties themselves observable.

You can get the underlying JavaScript array by invoking the `observableArray` as a function with no parameters, just like any other observable. But `observableArray` has equivalent functions to those of the underlying array, and for functions that modify the contents of the array (such as `push` and `splice`), the dependency tracking mechanism notifies all registered listeners of the change so that your UI is automatically updated.

### The `foreach` binding

Bindings within the `foreach` block can refer to properties on the array entries. To refer to the array entry itself, and not just one of its properties, use the special context property `$data`.

If there isn't an element on which you can put a `foreach`, `if`, `ifnot`, or `with` binding, you can use the containerless control flow syntax, which is based on comment tags.

When using `foreach`, the `afterAdd` callback is invoked when new entries are added. You can have it call jQuery’s `$(domNode).fadeIn()` to animate the transition. The `beforeRemove` callback is invoked when an array item has been removed, but before the corresponding DOM nodes have been removed. You can have it call jQuery’s `$(domNode).fadeOut()` to animate the transition. Because Knockout cannot know how soon it is allowed to physically remove the DOM nodes, if you specify a `beforeRemove` callback, then it is your responsibility to remove the DOM nodes.

### The `if` binding

The `if` binding physically adds or removes the contained marking in your DOM and only applies bindings to descendants if the expression is `true`. The `visible` binding just uses CSS to toggle the container element’s `visiblity`. The contained markup always remains in the DOM, and always has its `data-bind` attributes applied.

### The `with` binding

The `with` binding will dynamically add or remove descendant elements depending on whether the associated value is `null`/`undefined` or not.

### The `template` binding

Named template markup is wrapped in `<script type="text/html">`. The `type` attribute is necessary to ensure that the markup is not executed as JavaScript, and Knockout does not attempt to apply bindings to that markup except when it is being used as a template.

Using the `foreach` binding with a named template gives the same result as embedding an anonymous template directly inside the element to which you use `foreach`.

If you have multiple named templates, the `name` option can specify a callback function to determine which one of them is used. If using the `foreach` template mode, Knockout will evaluate the function for each item in your array, passing that item’s value as the only argument. Otherwise, the function will be given the `data` option’s value or fall back to providing your whole current model object.

### Loading and saving JSON data

* The `ko.toJS` method clones your view model’s object graph, substituting each observable with its current value, so the returned object has no Knockout-related artifacts.
* The `ko.toJSON` method converts the returned value from `ko.toJS` to a JSON string.

To have a live representation of your view model data for debugging, use:
{% highlight html %}
<pre data-bind="text: ko.toJSON($root, null, 2)"></pre>;
{% endhighlight %}

### Unobtrusive event handling

Anonymous functions were typically the recommended techinique to pass arguments to `data-bind` attributes, which leads to verbosity. Instead, event handlers can be attached unobtrusively by using jQuery's `on` method and the following Knockout helper functions:

* `ko.dataFor(element)` returns the data that was available for binding against the element.
* `ko.contextFor(element)` returns the entire binding context that was available to the DOM element.

## AngularJS

### Developer guide

The following are notes from [http://docs.angularjs.org/guide/](http://docs.angularjs.org/guide/).

#### Introduction

Angular features separation of data, application logic, and presentation components; data binding between data and presentation components; dependency injection; an extensible HTML compiler written in Javascript, and ease of testing.

Angular is designed primarily for developing single-page apps, and supports browser history, forward and back buttons, and bookmarking for them.

#### Overview

Angular teaches the browser new syntax through a construct we call directives, such as data binding with <code>&#123;&#123;expression&#125;&#125;</code>, DOM control structures, and grouping HTML into reusable components.

Angular is opinionated about how a CRUD application should be built; it has a seed application with directory layout and test scripts as a starting point.

#### Bootstrap

Angular initializes upon the `DOMContentLoaded` event, at which point it compiles the DOM, treating the `ng-app` directive as the root of the compilation.

#### HTML compiler

Angular's HTML compiler allows you to attach behavior to any HTML element or create new HTML elements or attributes with custom behavior. These behavior extensions are called directives.

The compilation phase collects all the directives; linking combines the directives with a scope to produce a live view.

Many templating systems perform one-way data binding, consuming data and a template in string form, producing a new string that is assigned to an element's `innerHTML`. Angular consumes the DOM with directives and produces a linking function, which combined with the scope produces a live view.

#### Conceptual overview

The view is a projection of the scope onto the template, or HTML. The scope is the glue which marshals the model to the view and forwards the events to the controller.

In Angular, the template is the HTML, not HTML with template embedded. The DOM rendered by the browser is passed into the Angular's compiler, which sets up watches on the model.

There is only one injector per application. If an object does not exist in its cache, the injector asks the instance factory to create a new one. A module is a way to configure the injector's instance factory, known as a provider.

Calling the `invoke` method on an injector and passing a method causes the injector to call that method with arguments derived from their names.

#### Directives

Directives can use either the characters `:`, `-`, or `_`, and can optionally be prefixed with `x-` or `data-` to make it HTML validator compliant.

The `$compile` method produces a linking function for each directive in the DOM, and returns them all in a combined linking function. Linking this with `scope` calls the individual linking functions, allowing them to register any listeners and set up any watches with the `scope`.

#### Expressions

Evaluation of all properties takes place against a scope. To refer to the `window` object, Angular expressions must use `$window`.

Expressions cannot contain control flow statements. If you need a conditional, loop, or to throw from a view expression, delegate to a Javascript method instead.

Angular augments existing objects with additional behavior, and prefixes its additions with `$` to avoid collisions.

#### Modules

Breakup your application into multiple modules: a service module (for service declaration), a directive module (for directive declaration), a filter module (for filter declaration), and an application-level module which has initialization code.

A module is a collection of configuration and run blocks. Configuration blocks get executed during the provider registrations and configuration phase, and only providers and constants can be injected into them. Run blocks get executed after the injector is created, and only instances and constants can be injected into them.

Modules simply manage the `$injector` configuration, and have nothing to do with the loading of scripts into a VM.

Typically an app has only one injector, but when unit testing, each test has its own injector.

#### Scopes

Both controllers and directives have reference to the scope, but not to each other. This isolates the controller form directives and the DOM.

When new scopes are created, they are added as children of their parent scope, thereby creating a tree structure that parallels the DOM where they're attached and uses prototypical inheritance.

Angular automatically adds the `ng-scope` class on elements where scopes are attached.

Scopes are attached to the DOM with the `$scope` data property, and the root scope is attached to the element with the `ng-app` directive.

Scopes can propagate events like the DOM does; `broadcast` will propagate to children, while `emit` will propagate to parents.

For scope mutations to be properly observed, you should make them only within the `scope.$apply()`, which Angular APIs do implicitly.

Observing directives like <code>&#123;&#123;expression&#125;&#125;</code> register listeners using the `$watch()` method. Listener directives such as `ng-click` register a listener with the DOM; when that listener fires, it executes the associated expression and updates the view using the `$apply()` method.

#### Dependency injection

By having `ng-controller` instantiate a controller class, it can satisfy all its dependencies without the controller ever knowing about the injector.

Factory methods used by an injector return services, which are then injected into controllers.

The recommended way of declaring controllers with DI is:

{% highlight javascript %}
var MyController = function(dep1, dep2) {
  ...
}
MyController.$inject = ['dep1', 'dep2'];
MyController.prototype.aMethod = function() {
  ...
}
{% endhighlight %}

### Tutorial

The following are notes from [http://docs.angularjs.org/tutorial/](http://docs.angularjs.org/tutorial/).

#### Step 0: Bootstrapping

Angular uses `name-with-dashes` for attribute names and `camelCase` for the corresponding directive name.

During the app bootstrap phase, the injector creates the root scope that becomes the context for the application's model, and then compiles the DOM starting at the `ngApp` root element, processing directives and bindings found along the way.

#### Step 2: Angular templates

The value for the `ng-controller` attribute names a function; expressions in the template refer to the properties of the `$scope` argument passed into this function.

The controller scope is a prototypical descendant of the root scope created when the application was bootstrapped.

#### Step 3: Filtering repeaters

A filter function applies its filter-by argument against child properties of the expression being filtered.

The `ngBind` or `ngBindTemplate` directives are invisible to the user while a page is loading.

A `pause()` statement in the end-to-end test lets you explore the state of the application while displayed in the browser.

#### Step 4: Two-way data binding

If a `select` element is bound to a missing property in the model using directive `ngModel`, then Angular will temporarily add a new "Unknown" option.

#### Step 5: XHRs and dependency injection

To use a service in Angular, declare the names of the dependencies you need as arguments to the controller's constructor function.

Don't use a `$` prefix when naming your services and models in order to avoid collisions.

The dependency injector cannot infer dependencies correctly if you minify your Javascript code; to fix this, explicitly assign service identifier strings to the constructor's `$inject` property.

The mock `$httpBackend` service will only resolve its returned promises, or return a response, when its `flush` method is called.

#### Step 6: Templating links & images

The `ngSrc` directive prevents the browser from treating the Angular expression as literal markup and issuing a request to an invalid URL.

#### Step 7: Routing & multiple views

The injector loads module definitions, registers all service providers in those modules, and when asked to inject a function with a service, creates one from its provider.

To configure your application with routes, you must create a module for your application.

The `ngView` directive includes the view template, or partial template, for the current route into the layout template.

#### Step 9: Filters

To create a new filter, you must create a new module and register your custom filter with the module.

#### Step 11: REST and custom services

The `factory` method of the module API registers a custom service using a factory function.

The `$resource` service is easier to use than `$http` for interacting with data sources exposed as RESTful resources.

A service may look like it returns results synchronously, but instead a future is synchronously returned that the XHR response fills with data.

The `toEqualData` Jasmine matcher compares two objects, but takes only object properties into account and ignores methods.
