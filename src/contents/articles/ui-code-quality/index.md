---
title: How to improve UI code quality in micro services architecture
author: Vamshi Krishna
date: 2016-07-12 10:02:00 +05:30
template: article.jade
comments: true
---

### Introduction
The UI code consists of client side code ( Javascript , HTML, CSS). This document talks about the problems faced in UI code quality management and few suggestions on how to fix them. Being micro service archtecture adds more complexity to the issue. We need to figure out work arounds to handle these problems.

The document is going to be in Question & Answer mode , where the question describes the problem being faced and the answer the solution. Sometimes the solution might create more problems, so the subsequent question might be based on the solution given to previous question. 

The context of the project is going to be UI code with following frameworks and libraries: 
- Angular 1+
- grunt
- jade 
- Coffeescript, javascript, ES2015 co-existing
- jquery in legacy code 

---

### Why UI Code quality is important
The share of UI code in web applications has increased considerably over the past couple of years. The importance of standardizing and improving the UI Code increased accordingly. 
Few years back there was no concern related to UI code quality, but these days it has become an important concern due to the following reasons : 
- Share of UI code drastically increased over the couple of years. The logic once resided in backend is now moved to frontend.
- Forgiving nature of Javascript and HTML. There are no strict rules or compiling errors which makes very difficult to find errors
- Uncontrolled evolution of UI code. The language and framework are upgraded pretty fast in recent years. The code need to be well organized and well tested to be able to upgrade to newer technology.

---
### Challenges faced in microservices ( and UI code in general )
Microservices add more challenges to already existing challenges in UI code.
- Legacy code 
  - Different projects using different frameworks
  - Older code with different framework like jQuery 
  - Different format of javascript like coffeescript
  - Code without test coverage
  - Different practices were used in early stage of Angular 
- Different styles 
  - Javascript code can be written using functional style or object oriented or procedural 
  - Indentation preferences
  - Component based vs Controller based ( Angular )
- Reluctance to comply 
  - Deciding a style is impossible. Even if it is decided getting everyone comply to the rules is another challenge
- Task deadlines 
  - Its easy to skip best practices to reach deadlines as javascript is very forgiving 
---
### Eslint : Code quality assurance
Eslint gives a set of rules which can be configured. We can configure the build to fail when these rules are broken. Rules can be configured. Custom rules can be added. Rules can be configured using a configuration file .eslintrc.
Eslint helps in 
- Prevent trivial mistakes like code after return statement etc 
- Maintain consistency in code ( single quotes, characters limit, forcing ES2015 etc ) 
- Avoid antipatterns. When a bad coding practice is observed , we can write a custom rule to prevent that being written again in future
- We have framework specific rules which can be applied to enforce good practices
- Almost all editors have eslint plugins which gives instant feedback while developing 

## What is the problem ?
We have different configuration file for each project. This causes the following problems : 
- Each project will have its own rules configuration, no consistency
- Adding a new rule need to be added in each project. In our case where there are two digit number of projects, its a big hastle 
- Few old projects use jshint , which need to be replaced with eslint 


## Solution ? 
We can have a single file shared by all projects. We have a single project which defines the rules and all other projects import from this configuration. The way to do this is by having a project hosted publicly. 
```js
//index.js in base-eslint-config project
module.exports = {
  'env': {
    'browser': true,
    'jasmine': true,
    'node': false,
    'amd': false,
    'mocha': false
  },
  'ecmaFeatures': {},
   'globals': {
    'angular' : true,
    'moment' : true,
     //for tests
    'inject' : true,
    'module' : true
  },
  'rules': {}
  };
```

This configuration can be installed into each project by 
```
npm install --save https://path-to-repository
```
This installs the configuration into our project. We can then use all the configurations mentioned there just by extending it. 
```js
//.eslintrc
{
"extends": "base-eslint-config",
  "plugins": ["angular"],
  "rules": {
    "no-var": 0,
    "angular/di-unused": 2,
    "angular/no-http-callback": 2
  }
}
```

- We can write additional rules to either override the base project rules or add new rules
- When we start using eslint, legacy code is not yet compliant. We can add rules to disable warning and keep enabling them one by one as and when we refactor the code to fix the eslint errors

---

### Build : Sharing build scripts 

We use build scripts while developing UI code . Mostly the tasks are: 
- Check for eslint errors 
- Transpile ES2015,coffeescript,typescript etc to plain javascript 
- Run unit tests 
- Convert jade to html 
- Convert sass/scss to css 
- Combile code in multiple files to single file

At present we use Grunt to manage the build tasks. Grunt tasks are maintained using Gruntfile.js .

## Problem ? 
Each project has its own Gruntfile. Which is very problematic for the following reason : 
- Very difficult to add support for new tool ( like eslint, babel, code coverage etc., ). We need to go to each project and edit the script manually . 
- Inconsistent practices. Some projects have optimized scripts while older projects have slow or unfunctional scripts. Difficult to hold the project task and fix the issue. 

## Solution : 
Again, use a single grunt config and share it across. The way to do is using node app [Croak](https://github.com/AdesisNetlife/croak). With croak, we can have single project with all the grunt configs enabled. Each project will have a croak config instead of Grunt and we can override the rules declared in gruntfile. 

```js
// example from croak documentation . 
module.exports = function (croak) {
 
  croak.extendConfig({
    log: {
      foo: {},
      bar: 'hello croak'
    },
    read: {
      files: [
        '<%= croak.root %>/file.json'
        __dirname + '/another-file.json'
      ]
    }
  })
 
  // you can also register new tasks if 'register_tasks' option is enabled 
  croak.registerMultiTask('log', 'Log stuff.', function() {
    grunt.log.writeln(this.target + ': ' + this.data)
  })
 
  croak.registerTask('default', ['log'])
 
  }
```

## Using node (instead of Grunt/Gulp) 
We can write our own script custom build for our requirements. These are few node apps which can be used : 

|Purpose                | Node app
|-----------------------|:----------------
|Watching file changes  | chokidar
|Transpile ES2015       | babel
|Transpile coffeescript | coffee-script
|jade/pug               | jade
|file read/write        | shelljs
|minify                 | uglify-js


A simple build script looks like this : 

```js 
//build.js
const watch = require('chokidar').watch;
const babel = require('babel-core');
require('shelljs/global');
const presets = {
  presets: [ 'es2015']
};

watch('./src/js/**/*.js')
  .on('add', (path) => {
    mkdir('-p', './temp'+path);
    transpileToEs5(cat(path)).to(destinyFile);
  })
  .on('change', (path)=> {
    mkdir('-p', './temp'+path);
    transpileToEs5(cat(path)).to(destinyFile);
  });

''.to('./build/site.js'); //empty the file everytime the build is run

watch('./temp/**/*.js')
  .on('add', (path) => {
    cat(path).toEnd('./build/site.js'); //append to the end of the file 
  })
  .on('change', () => {
    cat('./temp/**/*.js').to('./build/site.js'); //override whole file
  });

function transpileToEs5(code){
  return babel.transform(code, presets);
}
```

This script can be imported to all projects by using exports/imports.Different paths and other variables can be injected through a config file. This way , when we want to add a new feature, we just need to add that to this script and we have the feature available to all projects. 

### Testing ( and Code coverage )
Words cannot describe the importance of testing in quality assurance. We have two types of testing : 
- Unit testing using Angular and Jasmine
- e2e testing using Protractor and Jasmine

## Unit testing 
Unit testing is testing a given function or module by mocking all other interactions. If the module calls another service or module, we mock the response of that module so that we are concerned only about the code we are testing. Unit tests are very light due to mock data and are thus very easy to run. Most of the cases every change in our code is accompanied by running all the related unit tests to make sure no other functionality broke. 

We use karma for testing jasmine tests. The Gruntfile has related entry : 
```js
{
        options: {
          files: [
            '<%= test_files.vendor %>',
            '<%= vendor_files.js %>',
            '<%= app_files.js %>',
            '<%= test_files.js %>'
          ],
          browsers: ['PhantomJS'],
          logLevel: 'DEBUG',
          frameworks: ['jasmine'],
          preprocessors: {
            'src/**/*.js': ['babel'],
            'src/**/!(*.spec|*.mock)*.js': ['coverage']
          },
          coverageReporter : {
            reporters:[
              {type: 'html', dir:'build/coverage/', subdir: '.'},
              {type: 'text-summary'}
            ]
          },
        },
        unit: {
          runnerPort: 9101,
          port: 9103,
          background: true
        },
        continuous: {
          singleRun: true
        }
      }
```
Code coverage : Code coverage is a metrics which shows what percentage of code is handled in unit tests. It even calculates the coverage in conditional branches. 

The entry `coverateReporter` in the above script validates the unit tests and generates html files which give detailed information on total code coverage . It shows code coverage at each line level. We can have build configured in teamcity,travis-ci etc to read these docs and break build if total code coverage percentage is lesser than previous build. 

## e2e testing 
e2e ( End to End ) testing is automating real actions . We do not mock any data . Its as if a human opens the url in browser, inputs data and checks the page is behaving as expected. 
The page does the network call to fetch data and everything is realtime. For this reason, e2e is very expensive operation. We have data and network overhead. Its slow and resource intensive. But as it is not a simulation but real job automated , its very valuable. We check the e2e tests only when we are ready to push the code unlike unit tests which are checked after each change . 

The general convetion for e2e tests are to have `po` objects(files) and spec files.`po` files contains objects which have information on different pages of the webapp like url, fields, navigation details etc., `spec` files interact with `po` files to get info and execute tests. 

Example: 
```js 
//login-ui po 
var LoginPage = function() {
  var userName = element(by.id('userName'));
  var passWord = element(by.id('password'));
  var signInBtn = element(by.id('sign-in'));
  
  // url of Kareo
  this.getURL = function() {
    browser.get('...'); //fill in url 
  };

  // set username
  this.setUserName = function(username) {
    userName.sendKeys(username);
  };

  // set password
  this.setPassWord = function(password) {
    passWord.sendKeys(password);
  };
  
  // click sign in button
  this.clickSignIn = function() {
	  signInBtn.click();
  };
  
  // get title of the browser
  this.browserTitle = function() {
	  return browser.getTitle();
  };
  
};

module.exports = LoginPage;
```
```js
//spec.js

var LoginPage = require('../po/login-page.po');
describe('login page', function() {
  it('Login to Kareo', function() {
    var loginPage = new LoginPage();
    loginPage.getURL();
		expect(loginPage.browserTitle()).toEqual('Login | Kareo');
	
    loginPage.setUserName('..'); //fill in user id 
    loginPage.setPassWord('...'); //fill in password
    loginPage.clickSignIn();
		// verify search Text box after login 
  });
});
```

## Problem :
Every project need to use login PO to login into the app before navigating to their project. Similarly there are many other PO objects which need to be shared across the projects. 
As all projects are private repositories , we can not share the code as we did earlier. 
## Solution :
The probable solution for this is to have PO objects added to a subrepo. We are using hg (mercurial) so the code snippet will be for mercurial. 
Create a new repository for login-po , lets assume it is https://bitbucket.org/org/login-ui-po 
We perform the following in our login-ui project. 
```
$ cd login-ui 
$ echo 'test/po = https://bitbucket.org/org/login-ui-po' > .hgsub
$ hg add .hgsub
$ hg commit -m "Adding subrepo" 
$ cd test
$ hg clone https://bitbucket.org/org/login-ui-po po
```

We are adding a subrepo to our login-ui by adding the info in the file `.hgsub`. We need to provide info in the form of 

&lt;destination folder&gt; = &lt;repository url&gt;

Then we clone the repository in the correct path. Now we can clone login-ui-po into all projects which need login-ui `po` objects. Adding code to `login-ui-po` will be reflected in all projects which are using it . When we perform pull for our project, the subrepo will also get updated . This way we are linking po objects in multiple projects when and where they are needed without changing any other build process or structure. 


