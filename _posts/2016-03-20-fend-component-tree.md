---
layout: post
keywords: blog
description: blog
title: "Component interaction and using RxJs/observable"
categories: [frontend]
tags: [javascript, es6, angular, reactive]
icon: code
---
> 此篇文章是在下为公司内部交流所写。整理后放入私人博客供参考。

## Component based APP
- each major component is made with smaller components. In angular 1.4, we simulate components with directives and later in Angular 1.5 and 2.0 we will have standard component API
- each component should only control its own view and data
- each component (or a directive) should has its own set of bound input and output, like a well-defined public API. The parent and child talk to each other with the input and output functions or data.

This makes each component as a blackbox following the golden rule **separation of concerns**.

Here is a good article in Angular 1.5 official docs on [Component-based application architecture](https://docs.angularjs.org/guide/component).

### Component interaction
The rules of using API for component interaction looks promising at first look, but we will soon have problems with it. Below is a common component tree of a complex view. Think that each box is a component (and each of them may also be consisted with smaller components).

![A complex component based app](/image/post/complex_component.png)

There are various ways of doing component interactions in a complex app with angular 1.x. One problem is that we don’t have a best practice guideline on how to do data flow and event handling. To name a few possible ways:

1. emit/broadcast event on `$scope` chain. This may be the most common way of event handling in our app right now. The issues of using it:
- scope object will be removed from Angular 2, which indicates that using this way is no longer encouraged
- we need to be careful for the event flow. For example, by default an event will be broadcast from main to bottom and each controllers on the way will probably handle the event, but may be we just want to handle it once
- it’s not possible to communicate between sibling components using this way. for example, we cannot pass an event from top to bottom, or between left and right. In the legacy app, we have a ChannelService to make event reflects in the scope hierarchy. In my opinion this solution is a bit hacky and hard to maintain
2. introducing an angular factory to store a global state object and assign its reference to the directive’s controller, thus we could use the directive’s scope to bind watchers to the global state. However:
- the watcher functions should be idempotent, otherwise we cannot guarantee termination of angular digest cycle
- the watcher will always run when initializing, however the context may not be ready yet and there will be some dependency issues
- the global state can be modified by any subject easily and the maintainer may not be aware that if there are side-effects modifying them.
3. we have no context why the value changes when a watcher is triggered.
require the parent controller, thus the child controller can have access to the parent’s controller object in the link hook.
- since we add new dependencies, the reusability will be limited
4. params passing via isolate scope. In angular 1.5, we can have one way binding to ensure one way binding. This looks like a standard way of defining input/output API.
- it is easier to communicate between adjacent parent-child with this approach, but:
- will be a lot harder if we want to pass data from Main to a deeper component such as bottom, since Main cannot directly have access to the Bottom‘s API. If we enforce this rule, we may need to define the same APIs for each component in the way.
5. A basic Observer pattern implementation as a global Angular service. Since we often have prior knowledge on the possible events in the system, we can inject the event emitter to a global factory, so that each component can attach new listeners to the global event.
6. plain old DOM events
- Similar issues to the scope solution. We may not consider this solution since this is highly bound to DOM, which is not a good practice to do in Angular.

#### Summary

Here are some of my suggestions after listing all kinds of these component interaction solutions:

- We probably should avoid using `$scope` for event handling from now on
- Use the component API interaction styles whenever possible
- Even though in a standard component based app, we may not need a global service for “long range” communication, however we always need some way to do it, especially between different major modules. For example, the activity log may need to listen to a global ‘open activity log’ event which may be emitted anywhere in the app.

In my opinion, what we need for now is a standalone messaging/component interaction data service to let the major components interact with each other. observer pattern could be a solution, but here I want to propose the use of RxJs into our app.

## Using Reactive.js

### What is Reactive and Observable

I don’t want to explain reactive programming deeply since this email could be a paper. You may find the following articles worth reading:

- [Reactive programming on wiki](https://en.wikipedia.org/wiki/Reactive_programming)
- [Functional Reactive Programming for Angular 2 Developers - RxJs and Observables](http://blog.jhades.org/functional-reactive-programming-for-angular-2-developers-rxjs-and-observables/)
- [The introduction to Reactive Programming you’ve been missing](https://gist.github.com/staltz/868e7e9bc2a7b8c1f754)
  - This is highly recommended to read.
  - It also provide a [EggHead course](https://egghead.io/lessons/javascript-introducing-the-observable) (not free)
- [Official Angular 2 docs on component interaction](https://angular.io/docs/ts/latest/cookbook/component-communication.html#!%23bidirectional-service)
- [Observable docs](http://reactivex.io/documentation/observable.html)
- [Principles of Reactive Programming](https://www.coursera.org/course/reactive)
  - It is an advanced course for functional programming in Scala.
- [ES7 Observable proposal](https://www.coursera.org/course/reactive)

Angular 2 also has built-in RxJs usages in its [http-client](https://angular.io/docs/ts/latest/guide/server-communication.html#!%23rxjs).


#### TL;DR

The key component of a reactive programming library is the `Observable` object. An `Observable` is essentially a data stream of values in time. In our case, the most important usage is to construct new global event streams, so that we could subscribe to them, and react to the new values.

#### Compared to plain observer pattern solution

The idea is quite similar to the observer pattern. However, the observer pattern is just an callback factory to be triggered sequentially when a new event emits. The `Observable` object also provides some other flexible functionalities:

- event streams can be combined and transform them with functional operators like map, filters to produce new Observable streams. Also this gives us the ability to separate the listeners in different places without inference each other
- unlike observer pattern, the event subscribe functions will be called asynchronously
- and more …

### Using in Angular 1.x

We could use RxJs/Observable in Angular 1.x system as an global Event manager service. Different components could create their own event streams (observables) and register in the manager with identifiers, which means we store all global event streams which may be used in different component in a key-value pairs map. So the subscribers could fetch the observables with keys, transform them into new Observables and then attach callbacks to the events.
