---
layout: post
title: AngularJS and RequireJS can live together
date: 2012-12-30T00:05:36+00:00
categories: [blog]
tags:
  - angularjs
  - javascript
  - programming
  - modules
---

*Note: I pulled this from an old abandoned site. It will be hopelessly out of date by now*

#### The Players

[AngularJS](http://angularjs.org/ "AngularJS Home Page") includes a module
system to help decouple code, but it stops short of locating the code in files.
The modules and the <a title="Dependency Injection">DI</a> help to take a lot
of rough edges off day-to-day code, but it can't get rid of the need to list
all your neat little JS files in a big lump at the end of your page.

[RequireJS](http://requirejs.org/ "RequireJS Home Page") uses the [AMD
API](https://github.com/amdjs/amdjs-api/wiki/AMD "AMD API Specification") to
find and load dependencies for modules (in this case, modules are 1&ndash;1
with files). It does it by surrounding all the code you write in a (sometimes
contentious) wrapper to define a factory function. The factories are invoked
once all dependencies are resolved.

#### The Conflict

The two systems can play nice together, despite the clash over the name
"module", but not quite in the default configuration. New AngularJS projects
usually bootstrap an "application module" and the internal mechanics of both
libraries leads to a situation where RequireJS hasn't produced the application
module in time for AngularJS to bootstrap it.

#### Simple Resolution

The simplest fix is to manually bootstrap Angular with the application module
once enough of the system has loaded. Using the `ng-app` directive causes the
auto-bootstrap to run, so remove it and replace all scripts with the require.js
script pointing at main.js. Now all JS files should contain AMD modules and the
bootstrap code in main.js is what starts the application running:

  1. Remove the `ng-app` directive and point Require at the main file.
  ```html
      <body>
        <div ng-view></div>
        <script data-main="js/main" src="js/lib/require.js"></script>
      </body>
  ```
    
  2. Use a shim to blend Angular into the AMD namespace
  ```javascript
      require.config({
        shim: {
          angular: {
            deps: ['jquery'],
            exports: 'angular'
          }
        }});
  ```
        
  3. Define the Angular application module as an AMD module
  ```javascript
      define(function(require){
        'use strict';
        return require('angular').module('myApp', [])
          .config(['$routeProvider', function($routeProvider) {
            $routeProvider.when('/', {
              templateUrl: 'views/main.html',
              controller: require('controllers/main')
            });
          }]);
      });
  ```
            
  4. Bootstrap the module once loaded.
  ```javascript
      require( ['angular', 'app'], function(angular, app) {
        angular.bootstrap(document, [app.name]);
      });
  ```
  main.js should just contain this and the RequireJS config. 

Note how the controllers can be referenced via `require()`. It's just as easy
to register services and the rest:

```javascript
mod.controller('LogViewCtrl', require('controllers/logview'));
mod.service('BackendService', require('services/backend'));
```
                
#### Lasting Solution

The recipe above solves one problem &mdash; referring to files instead of
globals &mdash; but script loaders can also be used for lazy loading. Any
application that uses the router will probably have separate top-level views
and could benefit from only loading the code for the current view.

[Plenty](https://github.com/AngularBoss/boss-router) of
[people](https://github.com/cmelion/angular-yepnope/)
[have](https://github.com/matys84pl/angularjs-requirejs-lazy-controllers)
[tackled](https://github.com/pdswan/angular-amd) this
[problem](http://btilford.blogspot.co.uk/2012/08/modularizing-angular-app-with-amd.html).
The solution is usually to delay the completion of routing until the controller
script has loaded. If you follow the links, you'll see that require isn't the
only game in town. I've heard rumours of integrations with `goog.require()`.

I had some fun looking for a more complete solution, and by fun I mean
rummaging through the guts of the [Angular
code](https://github.com/angular/angular.js/blob/master/src/auto/injector.js)
as well as comparing approaches taken by different people. I tried extending
the core providers to resolve AMD module references and a couple of other
dead-ends. And then I got tired of chasing the perfect solution: I don't need
lazy loading that badly. But I am fond of a bit of module structure.

