---
layout: post
title: 'AngularJS Virtual Scrolling - part 1'
date: 2013-02-16T21:04:56+00:00
categories: [blog]
tags:
  - angularjs
  - javascript
  - programming
  - modules
---

*Note: I pulled this from an old abandoned site. It will be hopelessly out of
date by now and some of the formatting didn't survive the transfer*

#### AngularJS Lists

[AngularJS](http://angularjs.org/ "AngularJS Home Page") lets you define your
own HTML dialect by adding elements, attributes or behavioural classes. The
big win for web developers is the ability to move view logic back into the
page and use custom UI elements as they would an `<input>` or an `<ol>`.
These new elements are called directives, but AngularJS also includes
interpolation and filters for templating.

One of the more interesting directives is the `ng-repeat`. It's really the
only directive you need to introduce collections (or looping if you think
procedurally). But as is usually the case in web development, presenting a
lot of data can result in a high overhead. Consider a simple log view widget
that maintains a long list of log events and displays them in a scrollable
list. This is trivial to achieve

```html
<div style="overflow: scroll; height:200px;">
  <ol>
    <li ng-repeat="event in eventLog">
      {% raw %}{{event.time}}: {{event.message}}{% endraw %}
    </li>
  </ol>
</div>
```

The result is that every entry in the `eventLog` array is rendered into a
list element even though the browser won't display it. As log messages build
up, this can become quite a burden &mdash; particularly because AngularJS
will be dirty checking each element.

#### Optimisations

There are two areas where we could save:

1. Trim old log messages
2. Render only those messages visible in the scroll window.

Being able to trim old log messages and keep only the most recent few hundred
puts a nice bound on the problem. But it is a solution specific to log
messages and depends greatly on the context. Exactly how much history is
enough?
 Rendering a subset in a traditional (server-centric) web UI usually takes the
form of a pager. A fixed number of elements is displayed at any one time and
the user can move through the whole set in chunks (pages). For changing data,
this is not a nice presentation mechanism: it is always possible that the
page boundaries will move in the time it takes to move to another page,
leading to the impression of duplicates or lost entries.

<img src="/assets/virtual-list.png"
      alt="diagram of a virtual list" 
      width="135" height="197"
      class="alignright size-full" />

For ordered, changing data like log messages, a scrollable window is really
the only way a user can feel comfortable browsing the set. A trick for
reducing the rendering, with a long pedigree in UI toolkit support, is
<em>virtual scrolling</em>. The user is presented with a scroll bar next to
the viewport in the usual way, but only those elements currently visible are
actually rendered. As an optimisation, a region either side of the viewport
is also rendered to reduce flickering.

#### Simplest Solution in AngularJS

The simplest solution is to trim the collection that the `ng-repeat`
directive is rendering. We can achieve this with a filter:

```javascript
angular.module('sf.virtualScroll').filter('sublist', function(){
  return function(input, range, start){
    return input.slice(start, start+range);
  };
});
```

To use this effectively, notice that angular will wave its magic wand over
the names we supply and bind to variables in the scope. So using variables as
arguments to the filter directive means we can change the values with other
components and it will just work.

```html
<div style="overflow: scroll; height:200px;">
  <ol>
    <li ng-repeat="event in eventLog|sublist:rows:offset">
      {% raw %}{{event.time}}: {{event.message}}{% endraw %}
  </ol>
</div>
```

Inside the magic wand, the expression passed to the directive is split into 2
parts: the value identifier `event` and the collection identifier
`eventLog|sublist:rows:offset`. The collection identifier is evaluated each
time angular performs dirty checking and compared to the previous value. So
only the visible range is even considered. If the collection changes, the
display is updated, and if the `offset` scope value changes, the apparent
position within the list is changed.

So far, we only have half a solution as we haven't provided a way for the
user to change the scroll position. We've ensured that the offset of the
array slice is bound to a scope variable. Now we need another element to
modify that variable. Any widget would do but, to make comparisons simpler,
let's use a native scroll bar.


#### A Scrollbar Widget for the Browser

To make a scroll bar that behaves like one attached to a scrolling viewport
(i.e. a `div` with `overflow: scroll`), we need to trick the browser into
drawing just that: a viewport with empty content. By controlling the height
of the empty content, we control the range of our scroll bar. The only
subtlety is getting the height of the content right.

If we have `N` rows (repeats), then we need to set our empty content to have
a height of `N x H` (`H` is the height of a single row. If we wanted to be
exact in making our scroller match that in a real viewport, we would have to
calculate the height of the content to the nearest pixel. A simple
multiplication in theory, but the row height needed may not be available
right away. There is a surprisingly long series of events that need to occur
before we can be sure we have a valid height and the way directives are
structured makes it hard to get to this point. But we don't actually need
that row height. We can be approximate.

<div style="position:relative; width:100px;height:100px; float:right; border:1px solid gray;">
  <p>small H</p>
  <div style="position:absolute; height:100%; width:1em;top:0;right:0;overflow-x:hidden;overflow-y:scroll">
    <div style="width:1px;height:120px;"></div>
  </div>
</div>


<div style="position:relative; width:100px;height:100px; float:right; border:1px solid gray;">
  <p>large H</p>
  <div style="position:absolute; height:100%; width:1em;top:0;right:0;overflow-x:hidden;overflow-y:scroll">
    <div style="width:1px;height:1000px;"></div>
  </div>
</div>


All the row height multiplier (`H`) does is determine how fast the scroller
moves. When using the keyboard to move scrollbars, you are actually moving
the content. So the browser tries to make the distance moved make sense (a
line or two usually). If we tell our dummy content that it has a very small
row height, then user interaction will feel a bit too speedy. Too big, and
the user has the opposite difficulty. In the examples, try to imagine you are
scrolling through a long list. Picking a number that looks reasonable for a
row of text seems to work well in most cases.

#### A Scrollbar Widget in AngularJS

Creating a widget in AngularJS means writing a directive. Before we get into
the directive code, let's define how it will be used:

```html
<div style="overflow: scroll; height:200px;">
  <div sf-scroller="y = 0 to eventLog.length" ng-model="slicePosition">
  </div>
</div>
```

The directive needs to know the axis (`x` or `y`) and the range of the scroll
bar (think progress bar). If we want to bind the actual position to a model,
then the existing `ng-repeat` directive will define the binding for us. The
model expression could go in the main directive expression, but a 3 term
expression is somewhat easier to deal with and the `ng-model` directive is
designed for just this purpose.


Let's start with a helper function to parse the range expression:

```javascript
function parseRangeExpression (expression) {
  var match = expression.match(/^(x|y)\s*(=|in)\s*(.+) to (.+)$/);
  if( !match ){
    // throw an informative Error.
  }
  return { axis: match[1], lower: match[3], upper: match[4] };
}
```

The new scroll bar directive (`sf-scroller`) is relatively trivial, it just
tweaks the DOM and sets up bindings, so only a post-link function is needed. As
it requires an expression for the range, it must be restricted to an attribute
(rather than as an element or class). This is the most common case for angular
directives, so the form of the definition can be a simple function:

```javascript
var mod = angular.module('sf.virtualScroll');
mod.directive("sfScroller", function(){
  //function parseRangeExpression ...
  return function(scope, element, attrs){
    // ...
  };
}); 
```

I'm not going to list the rest of the code here, so please have a look on <a
href="https://github.com/stackfull/angular-virtual-scroll">github</a> for all
the details, but all that remains for the link function is to set up the CSS,
the dummy content element and the 2-way bindings via watches and scroll event
listeners.

The results can be seen by running the demo with the source on <a
href="https://github.com/stackfull/angular-virtual-scroll">github</a>. This
will get updated as the code progresses, but the basic operation should remain
the same.

#### Simplest != Best

There are 2 obvious downsides to this simplistic approach:

* For the developer &mdash; it's a pain to use if you have to bring your own
  scrollbar. It can never be made to match a real scroll pane and you have to
  be really careful about which events you pass on. Note that on the <a
  href="http://demo.stackfull.com/virtual-scroll">demo page</a>, no attempt was
  made to pass mouse and keyboard events to the scroll bar from the pane.

* For the user &mdash; it doesn't look like it's actually scrolling. The UI
  element doesn't appear to scroll in the way native scroll windows do,
  instead, it jumps to discrete rows. The visual effect is quite jarring.

If we really want a virtual scroller to look like a native scroll pane, we are
going to have to use a native scroll pane. Not just for the scroll bar widget,
but to display the content. To make the scrolling virtual, sleight of hand is
used to add and remove list elements just as they are needed. If you want
examples of this working now, you have to look at the big grid widgets such as
<a href="https://github.com/mleibman/SlickGrid/issues/22">SlickGrid</a> and <a
href="http://www.datatables.net/blog/Introducing_Scroller_-_Virtual_Scrolling_for_DataTables">DataTables</a>.
<a href="https://github.com/angular-ui/ng-grid">ngGrid</a> is promising this
functionality soon too. 

In the next post, I will be exploring a way to make virtual scrolling work with the <code>ng-repeat</code> style of templating&#8230;

