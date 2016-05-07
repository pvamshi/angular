---
title: Service based architecture
author: Vamshi Krishna
date: 2016-05-05
template: article.jade
---

AngularJS code resides mostly in the following :
* Controllers
* Services
* Directives

The weightage depends entirely on the architecture. We can write whole app using only Controller or entirely using directives. This blog is about giving maximum weightage to services and using directives/controllers only for view rendering. How it is advantageous and how to overcome the hurdles.

For the sake of simplicity lets just compare Controller with Service. Controller vs Directive/Component (ng-1.5) is due for another blog.


### Why no Controller ?


#### Service is singleton, Controller is not
The story goes like this. Whenever we open a page, the controller required to render the view of the page is instantiated. That means the whole code in Controller is run ( this is important). Before doing that , all the services upon which the controller is dependent are instantiated( same, the whole code in each service is executed). But ... if the service is already instantiated, the instance is just passed (Singleton !). So if we open the same page in the same app 10 times, Service gets instantiated 1 time and controller code gets run all the 10 times . Imagine doing the same HTTP call every time you open the same page for some reason . Thats total waste.


#### Service can be injected
Ok, controller is a loner whom no one cares except for the page its written for. Not fair to expect to be reusable. Still, the more social our code is, the easier it is to maintain and reuse.


#### Easy to test
Services are simple Javascript functions, so its straight forward to test them. Unlike in controller, where we can test only those that are tied to `scope` ( there is a way to overcome this as mentioned [here](/angular/articles/test-private/))

### How to use service ( use cases)
#### Store data (cache if possible)
Let the data be stored entirely in service. Before performing HTTP call check if the data exists in service already and return that. This goes for a full artible on it (coming soon, keep your questions open) .

#### Util services
Use services to have util functions as much as possible. If the data object contains 10 fields, all of them might not be needed in controller. Sometimes the data needed is fusion of multiple data sources etc., So the flow would be like :
```
Controller --> Util Service --> Data Service
```
The goal is to make Controller as light as possible. Use it absolutely only for tying view data to model data. Its ideal to have these util services pure functions, which do not store any kind of data.

This article will be updated with more details. And there are many unanswered questions, need more examples etc., We will be seeing all of them shortly.
