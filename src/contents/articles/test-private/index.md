---
title: Testing private functions in Angular
author: Vamshi Krishna
date: 2016-05-05
template: article.jade
comments: true
---

>This is a workaround. There is no framework support.

tl;dr : [plnkr.co/edit/EGOOTDBUSuPltHDvEEKx](https://plnkr.co/edit/EGOOTDBUSuPltHDvEEKx?p=preview)


AngularJS by default is a very test friendly framework. For someone who is eager to try functional programming, these days I am writing a lot of tiny private functions and I want to test them independently. Ofcourse this is not possible with the current Angular framework. To overcome this problem I tried a work around and this blog is about that.

Lets start with a simple example:

```js
angular.module('app',[])
  .service('Service1',Service1);

function Service1(){
  this.getMessage = function(num1,num2){
    return 'The sum of the numbers is '+sum(num1,num2);
  };
  function sum(num1, num2){
    return num1+num2;
  }
}
```

I am not interested to test `getMessage` . I only want to test `sum` as that is where all the logic happening. My unit test file looks like this:

```js
describe('main tests',tests);
function tests(){
  var service;
  beforeEach(module('app'));
  beforeEach(inject(function(Service1){
    service = Service1;
  }));

  it("should return the sum",function(){
    expect(service.sum(4,5)).toBe(9);
  });
}
```

I want to get this working. But it gives error.

To get this working, I added another service which always gives `false` for `testEnabled` value. In general case, we never touch it, but we will set it to `true` during testing. This way we can convert private functions to public during test and leave them private in general cases.

```js
angular.module('app',[])
  .service('Service1',Service1)
  .service('Service2',Service2);

//Service2 code

function Service1(){
  this.getMessage = function(num1,num2){
    return 'The sum of the numbers is '+sum(num1,num2);
  };
  function sum(num1, num2){
    return num1+num2;
  }

  if(Service2.testEnabled){
    this.sum = sum;
  }
}
```

Now we can continue with unit tests.

```js
angular.module('app',[])
  .service('Service1',Service1)
  .service('Service2',Service2);


describe('main tests',tests);
function tests(){
  var service, service2;
  beforeEach(module('app'));
  beforeEach(inject(function(Service2){
    service2 = Service2;
    service2.enableTest();
  }));
  beforeEach(inject(function(Service1){
    service = Service1;
  }));

  it("should return the name",function(){
    expect(service.sum(4,5)).toBe(9);
  });
}
```

Works great !!

The same can be used for private functions in Controllers too.
