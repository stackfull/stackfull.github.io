---
layout: post
title: 'AngularJS Virtual Scrolling - part 2'
date: 2013-03-24T22:25:16+00:00
categories: [blog]
tags:
  - angularjs
  - javascript
  - programming
---

*Note: I pulled this from an old abandoned site. It will be hopelessly out of
date by now and some of the formatting didn't survive the transfer*

#### Recap

In the [previous](/blog/2013/02/16/angularjs-virtual-scrolling-part-1.html) post, we
saw how virtualising a scroll window can enable the presentation of much larger
data sets. I detailed a simple method to present a subset of data using the
same UI components as a real list, but we saw it fall far short of
browser-native scrolling.

We also saw that plenty of javascript libraries have solved this problem in the
form of data grids. But one of the features of AngularJS that (for me) reduces
the noise associated with other template solutions is the `ng-repeat`. This
directive can cause the duplication of any HTML element for each element in a
data set. Making it a truly general purpose means to presenting collections.

#### The `ng-repeat` Directive

The goal is simple: to create a drop in replacement for `ng-repeat` that only
renders visible items. In reality, we are going to have to add some parameters
and restrictions. In the absence of a scroll window, there should be no
difference to the behaviour. So let's examine the existing code for
`ng-repeat`. Follow along with [the 1.0.5 code on
github](https://github.com/angular/angular.js/blob/v1.0.5/src/ng/directive/ngRepeat.js).

The first thing to note is that the directive uses `transclude` and provides a
`compile`
[function](https://github.com/angular/angular.js/blob/v1.0.5/src/ng/directive/ngRepeat.js#L64).
Simpler directives will just supply a link function, but the repeat directive
needs access to the composite linking function which will link compiled
elements to new scopes for each item in a collection. This is better explained
in [the guide](http://docs.angularjs.org/guide/directive) under _"Reasons
behind the compile/link separation"_.

In essence, the linking function [parses the repeater
expression](https://github.com/angular/angular.js/blob/v1.0.5/src/ng/directive/ngRepeat.js#L67)
and [sets up a
watch](https://github.com/angular/angular.js/blob/v1.0.5/src/ng/directive/ngRepeat.js#L92)
to do all the hard work. The watch is there to add and remove child elements in
sync with the collection, but it is careful to track movements as well as
additions and removals. If you have an element representing item in the
collection, and that element has some DOM state (a good example is a form
element), then you don't want to see that element get deleted and re-created
just because the underlying item moved within the collection. This is not
trivial to manage and there are [open
issues](https://github.com/angular/angular.js/issues/search?q=ng-repeat) with
the current version (1.0.5 at time of writing).

After all the shuffling has been taken care of, the code to actually [add a new element](https://github.com/angular/angular.js/blob/v1.0.5/src/ng/directive/ngRepeat.js#L161) or [delete an existing](https://github.com/angular/angular.js/blob/v1.0.5/src/ng/directive/ngRepeat.js#L178) is relatively simple. The composite linking function supplied by the `$compile` service to the directive compile function will do the work to attach a new element to a suitable scope.

#### A Virtual `ng-repeat`

Now we have some tough decisions to make. `ng-repeat` deals with items moving
within a collection, but there are lots of difficult edge cases (such as
primitives). The main reason for do this is to preserve DOM state associated
with an item, but we are not going to be able to handle this when hidden
elements are getting destroyed. So tough decision number one is to restrict
content of the directive from having state in the DOM (no forms, no jQuery
plugins that keep state) - all state must come from the items in the
collection.

Another nicety of `ng-repeat` is that the collection can be an object and
elements will be rendered using `for(key in collection)`. In a virtual list, we
need to target a subset of the collection by index. This would mean iterating
through the entire object each and every time. Not impossible, but potentially
a source of pain in a directive targeted at large data sets. So for the first
version, we are going to restrict use to arrays. We could potentially add
support for objects later if it proves feasible. 

#### Towards a `sf-repeat` Directive

The theory of a virtual list has already been explained, but there are still
some refinements to make. When to discard, when to render, how much to render.
When we come to deal with ajax pages of data, we may want to deal with
fixed-size chunks of items and elements. For now, we'll start with a simple
high-water and low-water system so that each time the viewport is scrolled,
elements are discarded if they are past the high water mark (distance from the
visible viewport) and new elements are created if they are within the low water
mark. These water mark levels will need to be configurable, but we'll ignore
that wrinkle for now and pick numbers that sound nice.

<img src="/assets/virtual-list-moving.png" alt="diagram of a virtual list scrolling" width="218" height="197" class="alignright size-full wp-image-94" />

In order to make the calculations about where the water marks are and even to
determine the height of the content pane, we need to know the row height. This
is surprisingly difficult and we have to get all restrictive again. The problem
is that during the compile/link phase, there is no point at which we can decide
that the child rows are fully rendered. There is always the possibility that
some interpolated data could change the height just after we think we're done.
Our restriction will be that it must be possible to determine the row height
from css and that each element will be displayed as block level. Oh, and no
margins please! After the first row is linked in, we will examine for an
explicit `height` or, failing that, `max-height`. Or we guess.

The next dimension we need is the viewport size. A virtual list without a
viewport is just a list, so we should be able to handle that case. For normal
use, it must be left to the user to define the viewport size using CSS. Just as
with row height, we will have to go looking for a `height` (or `max-height`) at
link time.

#### To the Code

We already have a module to use from the
[previous](/blog/2013/02/16/angularjs-virtual-scrolling-part-1.html) post, so we can
add a directive to that. It would be nice if we could just override the
necessary parts of the `ng-repeat` directive, but AngularJS directives don't
work like that. So lets dive right in with the directive definition:

```javascript
var mod = angular.module('sf.virtualScroll');
mod.directive("sfVirtualRepeat", function(){
  return {
    transclude: 'element',
    priority: 1000,
    terminal: true,
    compile: sfVirtualRepeatCompile
  };
  // ...
}); 
```

The compile function is going to do a little bit more work up-front than the
`ng-repeat` equivalent: we can pre-parse the repeat expression (just as we did
in part 1 for the scroller).

```javascript
function sfVirtualRepeatCompile(element, attr, linker) {
  var ident = parseRepeatExpression(attr.sfVirtualRepeat),
      LOW_WATER = 100,
      HIGH_WATER = 200;

  return {
    post: sfVirtualRepeatPostLink
  };
  // ...
}
```

The link function evaluates the expression identifying the collection we are
bound to and sets up the watch, much like before, but this watch is simply on
the length of the collection along with variables describing the active range.
In addition, the link function sets up a listener for scroll events from the
viewport which will adjust the current active range. Angular will notice that
the watched variables have changed, and call the listener for the watch. 

The watch is (as ever) where all the action happens. The algorithm can be
boiled down to comparing the new active range with the existing and deciding
what to destroy and what rows to create. Once the decisions have been made, we
are very much back with `ng-repeat`: creating child scopes and linking them in.
The subtlety is that the child DOM elements need to be positioned within their
container. We could use absolute positioning, or adding a margin to the first.
I've gone with the margin.

#### The Result

The `sf-virtual-repeat` directive is part of the `sf.virtualScroll` module on
github in [a source
repository](https://github.com/stackfull/angular-virtual-scroll) and [a bower
component](https://github.com/stackfull/angular-virtual-scroll-bower) with just
the built artifacts.

I've yet to really thrash the code, so don't bet your project on it! Please
feel free to load up the github issues with requests as well as bug
notifications. My next step will be to make a usable log viewer component and
maybe look at offloading data to the server too.

By far the biggest problem with this `sf-virtual-repeat` directive is how many
restrictions it places on the elements you can include. Also the CSS has to be
pretty explicit for the container element.

