---
layout: post
title: Angular Directives Exposing State
date: 2013-05-12T20:26:39+00:00
categories: [blog]
tags:
  - angularjs
  - javascript
  - programming
---

It's quite easy to find advice on [Directives Talking to
Controllers](http://www.egghead.io/video/LJmZaxuxlRc) (tell the directive what
function to call) and [Directive to Directive
Communication](http://www.egghead.io/video/rzMrBIVuxgM) (one requires a
controller defined in the other)[*](#egghead). Controller to Controller
communication is usually [via
Services](http://docs.angularjs.org/guide/dev_guide.services.injecting_controllers).
Many directives expose state via the scope to elements defined further down the
tree (e.g. [`ng-repeat`](http://docs.angularjs.org/api/ng.directive:ngRepeat)).
But I've yet to see any advice on exposing state from a directive to the rest
of the world.

The approach I took in [`angular-virtual-scroll`](https://github.com/stackfull/angular-virtual-scroll) was to use the [`ng-model`](http://docs.angularjs.org/api/ng.directive:ngModel) attribute (if present) to point to a model that will hold all important state variables. See [here](https://github.com/stackfull/angular-virtual-scroll/blob/0.4.0/src/virtual-repeat.js#L151) if you're interested. I made the dangerous assumption that nobody would try to alter the state - surely they wouldn't.

<a name="egghead" href="http://www.egghead.io/">* Pretty much everything at http://www.egghead.io/ is golden</a>.
