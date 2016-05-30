---
title: Service centric architecture in AngularJS
author: Vamshi Krishna
date: 2016-05-15 10:02:00 +05:30
template: article.jade
comments: true
---

* [Summary](#summary)
* [Introduction](#introduction)
* [General Idea](#general-idea)
  * [Service as Data containers](#service-as-data-containers)
  * [Immutable Data](#immutable-data)
  * [Solutions](#solutions)
* [Advantages](#advantages)
* * *

### Summary
This article is an argument on merits and demerits of using service as the data  container and letting Controller call service for all its data needs. It tries to decrease the amount of code written in Controllers and shift that to Services. Services are being used to store data. We do it in two ways :
1. Store data in service, copy required data to controller. Add watcher/observer in controller to update data.
2. Store data in service and directly use service methods to access data in views

We discuss the problems faced and try to fix them.


* * *

### Introduction
In all practical cases AngularJS code mostly resides in controllers. Controllers are heavily monitored and pampered by the framework, which makes it heavy and bloated. By literally living in controller all the time we are multiplying the problems.
We use the following page to demonstrate. Each city has a link to open new view. Thus it has new controller which fetches weather info from internet.

<iframe style="width: 100%; height: 200px" src="http://embed.plnkr.co/C6NOvo" frameborder="0" allowfullscren="allowfullscren"></iframe>


The scope of this article is entirely to AngularJS 1.5+.

### General Idea
#### Service as Data containers
The data flow in AngularJS is as follows :
* Controller asks service to give data
* Service does a HTTP call and return the promise to Controller
* Controller waits till promise is resolved and adds returned data to scope

Essentially the data is transfered from backend to scope, service just acts as a medium of transfer.

![Data transfer in AngularJS](//i.imgur.com/8soKsIO.png)

We are trying to stop the data in the service itself and make controller access the data from the service whenever it needs.

![Data transfer in AngularJS](//i.imgur.com/0fArCPA.png)

#### Immutable data
The data stored in service is Immutable by the controllers. In other words controllers have read-only access. This way the data stays consistent and several controllers can use it. Controllers shall consume the service data and transform it according to their `View` needs.

![Data transfer in AngularJS](//i.imgur.com/aaYXtDJ.png)

When the service data needs to be updated (for instance a new record is added or existing record is updated), controllers can send request to the service and service can update the data itself. Other controllers can update their data accordingly.

#### Solutions
In this article we are discussing two ways of achieving the above.
* First one is to copy a part or transformed part or whole of the data in the controller. This is as described in the above images. This poses a problem that Controller is not aware when Service changes the  data. We will discuss it further in [detail](#solution-1)
* Second one is to directly access the service data by creating a reference of the service in the controller

![Data transfer in AngularJS](//i.imgur.com/Ye1vNs6.png)


### Advantages
#### Service is singleton
* Service being singleton gets initialized only once, so are the network calls and other logic.
* Controller gets executed everytime the view is rendered, so are the HTTP calls and data transformations.
* Pushing more code into service optimizes the amount of HTTP calls and the logic we do in it. It happens only once per app.
* If needed we can do further calls or more logic as and when needed. We get more controll on how frequent we make the calls


#### Service is reusable
* We can easily inject Service into other services or controllers
* Controller is confined only for its view

#### Testing is easier in Service
* Simple functions, straight forward to test
* Enforce pure functions and test without any HTTP mocks

<iframe style="width: 100%; height: 400px" src="http://embed.plnkr.co/LDH0yT" frameborder="0" allowfullscren="allowfullscren"></iframe>

In the above example we could easily test the `extractWeather` function without needing to mock the http and handle promises. The utility function could be tested easily.

#### More modular code
* Moving data and logic to service makes Controller empty. Easier to break it into more simpler components
* Otherwise we face challenges like :
  * Sharing data from parent to child controller/component
  *

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
