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
* First one is to copy a part or transformed part or whole of the data in the controller. This is as described in the above images. This poses a problem that Controller is not aware when Service changes the  data. We will discuss it further in [detail](#solution-1-observer-pattern)
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
  * When one component or controller updates the data, it needs to inform other
      component . For this we need to rely on `$emit` and we are opening a new
      can of problems
  * The same code is repeated in multiple components which can be moved to
      service.

### Solution 1 : Observer Pattern
* Service contains the data with getters and setters functions

```javascript
//file : modal-service.js
//ModalService code

var modals = [];
function addModal(modal) {
  modals.push(modal);
}

function getModal(idx){
  return modals[idx];
}

function getAllModals(){
  return modals;
}

```

* Service provides functionality to let controllers register for a modal change

```javascript
//file : modal-service.js
//ModalService code
self.observers = {
  add: [],
  update: []
};

function addModal(modal) {
  //http call to create
  //on success do the following
  modals.push(modal);
  self.observers.add.forEach(function(createObserver) {
    createObserver(modal);
  });
}
```

* All controllers interested to update the view based on data present in the service register with the service to notify them when the modal is added or updated

```javascript
//controller code
var vm = this;
 vm.modals = []; //or $scope.modals = [];

 ModalService.observers.add.push(function(addedModal){
   vm.modals.push(addedModal);
 });
```
#### Advantages (Solution 1 )
* All the modal related data resides in the service
* Multiple controllers can register with the service and as soon as one of the controller modifies data, other controllers can update their view
* Get rid of `$emit` and `$broadcast`. Using $emit pollutes all the controllers and the order of views is important and cannot be changed.
* Using observer pattern makes it very light
* We can decide which functions need to be observed and updated

#### Problems (Solution 1)

**Problem 1:** Lots of boilerplate code. We need to add lot of code in each service. Observers for each method etc.,

**Solution**:
 We can move all the boilerplate code to a single service and inject it to all the services. We can fetch the functions list and code to be executed after each function execution.
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

ModalService.$inject = ['ObserverService'];

function ModalService(os) {
   os.wrap(self);
   ...
```
Controller code remains the same. A plnkr is provided below with full working example.


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
ModalService.observe('addModal', $scope, function(addedModal) {
  vm.modals.push(addedModal);
});
```
** Problem 3: ** Even while using `Controller As` syntax, we still need to inject $scope. And the controller is still populated with code related to observer. For each method of service, we need to add an observer method. But probably this is fine considering the advantages we get. We can check the second solution if that makes better sense.

Before that , summarizing all the changes, the example is shown in the following plunkr. When we add a modal in a controller , it gets updated in other controllers:

<iframe style="width: 100%; height: 400px" src="http://embed.plnkr.co/JCHWMaUyR66YqO66WN83" frameborder="0" allowfullscren="allowfullscren"></iframe>


### Solution 2 : Data centric services
* Service code remains same as Solution 1

```javascript
//file : modal-service.js
//ModalService code

var modals = [];
function addModal(modal) {
  modals.push(modal);
}

function getModal(idx){
  return modals[idx];
}

function getAllModals(){
  return modals;
}

```

* Append the read methods of the service directly to the view, so it can access
    directly

```javascript
//Controller code 
var modal = this; 
modal.getAllModals = ModalService.getAllModals;
modal.getModal = ModalService.getModal;

```
* View is going to look like this:

```html
<div ng-controller="ModalController as modal">
  <div ng-repeat="modal in modal.getAllModals()">
    ...
  </div>
</div>
```
As we are directly calling the service methods in the view, when a data changes
in the service, the view knows it and automatically updates the html
accordingly.

* Updating the modal data is done using the regular controller to service calls

```javascript
//Controlle code 

var vm = this;

vm.addModal = function(){
  ModalService.addModal(vm.newModal);
}

```
#### Advantages ( Solution 2)
Along with the [advantages of solution 1](#advantages-solution-1-) we have
these additional advantages: 
* Minimal code in Controller ( only appending the services to view and write)
* Angular adds watchers by default to all views so no need to add extra
    watchers

#### Problems ( Solution 2)
** Problem 1 ** : ( very important) Watchers cause infinite http calls. 
This is probably only reason why this method should not be used without
understanding how it works. Imagine a code like this : 

```javascript
//ModalService code 

this.getAllModals = function(){
  $http.get('...').then(function(response){
    modals = response.data;
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
vm.getModals = function(){
  // service code which does http call
}

```

```html
<!-- View code-->
<div ng-repeat="modl in vm.getModals()">
</div>
```
Angular prevents this by giving an error but still its a crime to attach a
function which does http call to the view. 

** Solution ** : 
Initially the service do not have any modals. So, we need to do either of : 
* Call HTTP call when service loads 
* Do the HTTP call when get call is made for the first time

First solution is not feasible as we will be loading too much data even if its
not needed. Second way causes the problem we are discussing now. 
We can fix this by adding a condition in our get call. 

```javascript
this.getModals = function(){
  if(!modals || !modals.length){
    $http.get('')
         .then(function(res){
            modals = res.data;
          });
  }
  return modals;
};
```
This again causes similar issue, when we call this twice in succession, it does
another http call when a call is already in progress. So we add another check
for promise. 

```javascript
var getModalsPromise ;
this.getModals = function(){
  if(!modals || !modals.length){
    if(!getModalPromise){
      getModalPromise = $http.get('');
      getModalPromise.then(function(res){
          modals = res.data;
        });
    }
  }
  return modals;
};
```
The above promise can be used in some other method too if needed. So that even
if multiple service calls need the same http call, we are only creating one
promise.

**Problem 2** : Data is not sync with database. 

We fetched data intially once and we are updating the modal data in service
only when someone does a add or update call . 

```javascript

var modals = [];
var getModalsPromise ;
this.getModals = function(){
  ...
  return modals;
};

this.addModal = function(modal){
  $http.post('',{})
    .then(function(){
      modals.push(modal);
    });
}
```

Only when one of the controller/component calls the addModal we are updating
our modals with new modal. What happens when some other user who added a modal
to the database which, our modal service is not aware of ? There is a sync
issue that arises.

**Solution**: We can fix this in two ways : 
* Do a http call frequently. Use $timeout and perform http call based on
    severity every 5 min or 10 min . 
* Have a socketio connection, which updates the service whenever there is a
    data change in the backend

** Problem 3 **: Too much data in memory

In case the modal list is too high or there are lots of services. The memory
usage might get too large. 

**Solution** : To fix this issue we can use session storage to
store the data.


```javascript


var getModalsPromise ;
this.getModals = function(){
  var modals = sessionStorage.getItem('modals');
  if(!modals || !modals.length){
    if(!getModalPromise){
      getModalPromise = $http.get('');
      getModalPromise.then(function(res){
          sessionStorage.setItem('modals',res.data);
        });
    }
  }
  return modals;
};

this.addModal = function(modal){
  $http.post('',{})
    .then(function(){
      var modals = sessionStorage.getItem('modals');
      modals.push(modal);
      sessionStorage.setItem('modals',modals);
    });
}
```

Conclusion : 

