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
* [Solution 1: Observer Pattern](#solution-1-observer-pattern)
  * [Advantages](#advantages-solution-1-)
  * [problems](#problems-solution-1-)
* [Solution 2: Data centric services](#solution-2-data-centric-services)
  * [Advantages](#advantages-solution-2-)
  * [problems](#problems-solution-2-)
* * *

### Summary
This article is an arguement on merits and demerits of using service as the data  container and letting controller call service for all its data needs. It tries to decrease the amount of code written in controllers and shift that to services. We store data in service in two ways :
1. Store data in service, copy required data to controller. Add watcher/observer in controller to update data.
2. Store data in service and directly use service methods to access data in views

We discuss the problems faced in each and try to fix them.


* * *

### Introduction
In all practical cases AngularJS code mostly resides in controllers. Controllers are heavily monitored and pampered by the framework, which makes it heavy and bloated. By literally living in controller all the time we are multiplying the problems.

The scope of this article is entirely to AngularJS 1 (1.5+ to be precise) .

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
The data stored in service is made immutable by the controllers. In other words controllers have read-only access. This way the data stays consistent and several controllers can use it. Controllers shall consume the service data and transform it according to their `View` needs.

![Data transfer in AngularJS](//i.imgur.com/aaYXtDJ.png)

When the service data needs to be updated (for instance a new record is added or existing record is updated), controllers can send request to the service and service can update the data itself. Other controllers can update their data accordingly.

#### Solutions
In this article we are discussing two ways of achieving the above.
* First one is to copy a part or transformed part or whole of the data in the controller. This is as described in the above images. This poses a problem that controller is not aware when service changes the  data. We will discuss it further in [detail](#solution-1-observer-pattern)
* Second one is to directly access the service data by creating a reference of the service in the controller

![Data transfer in AngularJS](//i.imgur.com/Ye1vNs6.png)


### Advantages
Lets few advantages generic to both ways of achieving this. 
#### Service is singleton
* Service being singleton gets initialized only once, so are the network calls and other logic.
* Controller gets executed everytime the view is rendered, so are the HTTP calls and data transformations.
* Pushing more code into service optimizes the amount of HTTP calls and the logic we do in it. It happens only once per app.
* If needed we can do further calls or more logic as and when needed. We get more controll on how frequent we make the calls
* If multiple components used in the view or the same component used in
    `ng-repeat` having the logic and memory in controller makes the app
    sluggish. By pushing the logic and data to service, we are making
    components very light and more freedom to modularize the code


We use the following page to demonstrate. Each city has a link to open new view. Thus it has new controller which fetches weather info from internet. Observe the network calls made. Click on a city twice by going to different city and coming back.We can observe the same data fetched by another unnecessary network call. If we move that logic to service, we will save all these network calls.
<iframe style="width: 100%; height: 200px" src="http://embed.plnkr.co/C6NOvo" frameborder="0" allowfullscren="allowfullscren"></iframe>

#### Service is reusable
* We can easily inject Service into other services or controllers
* Controller is confined only for its view

#### Testing is easier in Service
* Simple functions, straight forward to test
* Enforce pure functions and test without any HTTP mocks

 <iframe style="width: 100%; height: 330px" src="http://embed.plnkr.co/LDH0yT" frameborder="0" allowfullscren="allowfullscren"></iframe>

In the above example we could easily test the `extractWeather` function without needing to mock the http and handle promises. The utility function could be tested easily.

#### More modular code
* Moving data and logic to service makes controller empty. Easier to break it into more simpler components
* Otherwise, if we write code in components controller we face challenges like :
  * Sharing data from parent to child controller/component
  * When one component or controller updates the data, it needs to inform other
      component . For this we need to rely on `$emit` and we are opening a new
      can of problems
  * The same code is repeated in multiple components which can be moved to
      service.

### Solution 1 : Observer Pattern
* Service contains the data with CRUD functions

```javascript
//file : model-service.js
//ModelService code

var models = [];
function addModel(model) {
  models.push(model);
}

function getModel(idx){
  return models[idx];
}

function getAllModels(){
  return models;
}

```

* Service provides functionality to let controllers register for a model change

```javascript
//file : model-service.js
//ModelService code
self.observers = {
  add: [],
  update: []
};

function addModel(model) {
  //http call to create
  //on success do the following
  models.push(model);
  self.observers.add.forEach(function(createObserver) {
    createObserver(model);
  });
}
```

* All controllers interested to update the view based on data present in the service, register with the service to notify them when the model is added or updated

```javascript
//controller code
var vm = this;
 vm.models = []; //or $scope.models = [];

 ModelService.observers.add.push(function(addedModel){
   vm.models.push(addedModel);
 });
```
#### Advantages (Solution 1 )
* All model related data resides in the service
* Multiple controllers can register with the service and as soon as one of the controller modifies data, other controllers can update their view
* Using observer pattern makes it very light
* We can decide which functions need to be observed and updated

#### Problems (Solution 1)

**Problem 1:** Lots of boilerplate code. We need to add lot of code in each service. Observers for each method etc.,

**Solution**:
 We can move all the boilerplate code to a single service and inject it to all the services. We can fetch the functions list and code to be executed after each function execution. In the following code we are appending post-execution code which will get appended to each function by default. 
```javascript

function abc(a, b, c) {
  return a + b + c;
}

abc = (function(func, postexec) {
  return function() {
    var re = func.apply(service, arguments); // service is service instance (this)
    postexec(); // executing the callbacks
    return re;
  };
})(abc, function() {
  console.log('executing all callbacks registered'); //iterate over callbacks and execute them
});


```
Service code concedes to one line code

```javascript

ModelService.$inject = ['ObserverService'];

function ModelService(os) {
   os.wrap(self);
   ...
```
Controller code remains the same. A plnkr is provided at the end of section with full working example.


**Problem 2 :** When we keep adding the controllers to service observers, they keep getting accumulated. We need to remove the controller when view is changed ( controller is inactive).

**Solution**
we use `$destroy` to remove the observer in the `ObserverService` code.

```javascript
scope.$on('$destroy', function() {
  this._observers.splice(addedIndex, 1);
});
```
For this we need to pass the `scope` to the service. With this the controller code becomes :

```javascript
ModelService.observe('addModel', $scope, function(addedModel) {
  vm.models.push(addedModel);
});
```
** Problem 3: ** Even while using `Controller As` syntax, we still need to inject $scope. And the controller is still populated with code related to observer. For each method of service, we need to add an observer method. But probably this is fine considering the advantages we get. We can check the second solution if that makes better sense.

Before that , summarizing all the changes, the example is shown in the following plunkr. When we add a model in a controller , it gets updated in other controllers:

<iframe style="width: 100%; height: 400px" src="http://embed.plnkr.co/JCHWMaUyR66YqO66WN83" frameborder="0" allowfullscren="allowfullscren"></iframe>


### Solution 2 : Data centric services
* Service code remains same as Solution 1

```javascript
//file : model-service.js
//ModelService code

var models = [];
function addModel(model) {
  models.push(model);
}

function getModel(idx){
  return models[idx];
}

function getAllModels(){
  return models;
}

```

* Append the read methods of the service directly to the view, so it can access
    directly

```javascript
//Controller code 
var model = this; 
model.getAllModels = ModelService.getAllModels;
model.getModel = ModelService.getModel;

```
* View is going to look like this:

```html
<div ng-controller="ModelController as model">
  <div ng-repeat="model in model.getAllModels()">
    ...
  </div>
</div>
```
As we are directly calling the service methods in the view, when the data changes
in the service, the view knows it and automatically updates the html
accordingly. This is possible because angular adds the function to watchers
list and keeps checking if the returned data of the function changed. 

* Updating the model data is done using the regular controller to service calls

```javascript
//Controlle code 

var vm = this;

vm.addModel = function(){
  ModelService.addModel(vm.newModel);
}

```

The basic example is shown in this plnkr
<iframe style="width: 100%; height: 400px" src="http://embed.plnkr.co/HQVmSecqjWVT45FRO8z3" frameborder="0" allowfullscren="allowfullscren"></iframe>

#### Advantages ( Solution 2)
Along with the [advantages of solution 1](#advantages-solution-1-) we have
these additional advantages: 
* Minimal code in Controller ( only appending the services to view and write)
* Angular adds watchers by default to all views so no need to add extra
    watchers
* Controllers wont store data. Being very light they can be modularized and
    played around

#### Problems ( Solution 2)
** Problem 1 ** : ( very important) Watchers cause infinite http calls. 
This is probably only reason why this method should not be used without
understanding how it works. Imagine a code like this : 

```javascript
//ModelService code 

this.getAllModels = function(){
  $http.get('...').then(function(response){
    models = response.data;
  })
}
```
When this method is directly accessed in view, it is by default added to
watcher and for every single event this method gets called and we get infinite
http calls. Ofcourse this is true even if we are using the regular controller
method in the view like this : 

```javascript
//controller code 
var vm = this;
vm.getModels = function(){
  // service code which does http call
}

```

```html
<!-- View code-->
<div ng-repeat="modl in vm.getModels()">
</div>
```
Angular prevents this by giving an error but still its a crime to attach a
function which does http call to the view. 

** Solution ** : 
Before jumping to solution, lets see why this scenario occurs. Initially the service do not have any models. So, we need to do either of : 
* Call HTTP call when service loads 
* Do the HTTP call when get call is made for the first time

First solution is not feasible as we will be loading too much data even if its
not needed. Second way causes the problem we are discussing now. 
We can fix this by adding a condition in our get call. 

```javascript
this.getModels = function(){
  if(!models || !models.length){
    $http.get(url)
         .then(function(res){
            models = res.data;
          });
  }
  return models;
};
```
This again causes similar issue, when we call this twice in succession, it does
another http call when a call is already in progress. So we add another check
for promise. 

```javascript
var getModelsPromise ;
this.getModels = function(){
  if(!models || !models.length){
    if(!getModelPromise){
      getModelPromise = $http.get('');
      getModelPromise.then(function(res){
          models = res.data;
        });
    }
  }
  return models;
};
```
The above promise can be used in some other method too if needed. So that even
if multiple service calls need the same http call, we are only creating one
promise.
The code might look a littel huge and small mistake might cause bugs. So, we
can move this logic to another service and inject in each service as shown in
Solution 1 . 

**Problem 2** : Data is not sync with database. 

We fetched data intially once and we are updating the model data in service
only when someone does a add or update call . 

```javascript
var models = [];
var getModelsPromise ;
this.getModels = function(){
  ...
  return models;
};

this.addModel = function(model){
  $http.post('',{})
    .then(function(){
      models.push(model);
    });
}
```

Only when one of the controller/component calls the addModel we are updating
our models with new model. What happens when some other user who added a model
to the database which, our model service is not aware of ? There is a sync
issue that arises.

**Solution**: We can fix this in two ways : 
* Do a http call frequently. Use $timeout and perform http call based on
    severity every 5 min or 10 min . 
* Have a socketio connection, which updates the service whenever there is a
    data change in the backend

** Problem 3 **: Too much data in memory

In case the model list is too high or there are lots of services. The memory
usage might get too large. 

**Solution** : To fix this issue we can use session storage to
store the data.


```javascript
var getModelsPromise ;
this.getModels = function(){
  var models = sessionStorage.getItem('models');
  if(!models || !models.length){
    if(!getModelPromise){
      getModelPromise = $http.get('');
      getModelPromise.then(function(res){
          sessionStorage.setItem('models',res.data);
        });
    }
  }
  return models;
};

this.addModel = function(model){
  $http.post('',{})
    .then(function(){
      var models = sessionStorage.getItem('models');
      models.push(model);
      sessionStorage.setItem('models',models);
    });
}
```

### Conclusion : 
For very simple app, the usual Controller-centric way of coding is good. But in
case of complex apps, its always good to be more modular and keep the
controllers simple. Make use of services more than they are intended to and fix
the problems with workarounds mentioned above. 
