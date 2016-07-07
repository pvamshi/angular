---
title: Project building using node
author: Vamshi Krishna
date: 2016-07-06 18:02:00 +05:30
template: article.jade
comments: true
---
### Introduction
The de facto way of watching and building a UI project is by using either of grunt, gulp or rarely npm. In this blog we will see how it can be done using node alone.  

### General Idea
We will be using node apps to perform these actions. Depending on grunt ( I will be talking mostly about grunt as that is what being used in our project) might look simpler but writing our own script is easier to debug, maintain and scale over long run. The tasks we perform are going to be watch filesystem for changes and perform an action when there is a change. 

### Start 
Lets start the project from scratch. We can use existing project as well. The structure is going to be like : 
```
└── test
    ├── build
    │   ├── index.html
    │   ├── site.css
    │   └── site.js
    ├── node_modules
    ├── package.json
    ├── src
    │   ├── css
    │   ├── index.html
    │   └── js
    └── temp
```
`src` is where the source code is going to saved. `build` is for final build where all the javascript is loaded to single javascript file and all the css( sass comipled) in a single css.  We will be having all our javascript in ES6 ( can be typescript or coffeescript or other format too )  .
`temp` folder is where we store the compiled files ( js or css). 

The steps are as follows :

```shell
$ mkdir test 
$ cd test
$ npm init
$ mkdir src build temp src/js src/css
# touch src/index.html
#
```
When we run `npm init` the `package.json` file gets created after asking few basic questions. 
With this we have the basic file system ready.


### Watch 
To watch the file system for changes we use the package called [Chokidar](https://github.com/paulmillr/chokidar).
```shell
$ npm install --save chokidar
```
This adds a dependency in `package.json` along with installing it into `node_modules`. 
Create a build script to run our app. Lets call it `index.js` . 
```shell
$touch index.js
```
We can start using chokidar to watch files. 
```javascript
// index.js
const chokidar = require('chokidar');

chokidar.watch('./src/js/**/*.js')
  .on('add',(path) => {
    console.log('file added to watch '+path);
    })
  .on('change', (path)=> {
    console.log('file changed '+path);
  });
```
With this tiny code we have established a watcher on our file system. `**/*.js` matches all the files which end with .js in the particular file system ( /src/js in our case) . As we will not be using any other function of chokidar we can simplify the require statement a little : 
```javascript
const watch = require('chokidar').watch;

watch('./src/js/**/*.js')
  .on('add',(path) => {
  ...
```

### Read Write to files 
We can use default node apps to read and write to file system. But I found [shelljs](https://github.com/shelljs/shelljs) much easier to work with. With shelljs, the code is going to look exactly like shell but with functions instead of commands. Shelljs gives almost all important shell commands like cat, mkdir, echo etc ., 
Two ways to use shelljs , if we import shelljs/global we can directly access all the shell command ( cat(file)) otherwise we need to call it with a variable (shell.cat(file)). I liked the global format so will go with it. 
```
$ npm install shelljs --save
```
update the index.js
```js
//index.js
require('shelljs/global')
echo('echo test');
```
This should print the text to console . This is just to test if its working fine. 

### Transpiler for ES6
Assuming we will be writing our web app in ES6, we need to transpile the code to ES5. Lets use [babeljs](babeljs.io) for this task. 
```
$ npm install --save babel babel-preset-es2015
```
Lets convert some javascript from ES2015 to javscript 5
```js
//index.js
const babel = require('babel-core');
const presets = {
  presets: [ 'es2015']
};
function transpileToEs5(code){
  return babel.transform(code, presets);
}
echo(transpileToEs5(`let a = 5;`));
```
This should output `var a = 5;` to the output. 

### Final code
Lets combine all these to final code. 
```js
//index.js

const watch = require('chokidar').watch;
const babel = require('babel-core');
require('shelljs/global');
const presets = {
  presets: [ 'es2015']
};

watch('./src/js/**/*.js')
  .on('add', (path) => {
    console.log('file added to watch '+path);
    let code = cat(path);
    let es5Code = transpileToEs5(code);
    let destinyFile = './temp'+path;
    mkdir('-p', destinyFile);
    es5Code.to(destinyFile);
  })
  .on('change', (path)=> {
    console.log('file changed '+path);
    let code = cat(path);
    let es5Code = transpileToEs5(code);
    let destinyFile = './temp'+path;
    es5Code.to(destinyFile);
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
Here we are watching on `src` and `temp`. Whevever there is a change in src, we compile and copy to temp. Whenever there is change or new file added to temp directory, we copy whole of temp again to site.js. 
This is very basic code which can be further optimized a lot but left as is to make it easy to understand. 
We can add more watchers and more logic in each watch. 
That is it. If we write code in our src , it automatically compiles and updates site.js. 

### Conclusion
As we can see its very easy to write our own build code. It is very extensible, when we need to add more features , we just add new dependency and add another function in our code. Here are the advantages of using this instead of grunt or gulp: 
* Add or remove features easily
* Add customized features easily as its plain javascript code
* If we are dealing with micro services or multiple projects, We can add a config file and decide which ones to enable and disable for each project. All the projects can use the same script and its easy to impose rules or overwrite rules
* Convert this to a module and share across teams

