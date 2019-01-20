---
layout: post
title: Trees in AngularJS
date: 2014-02-09T21:30:57+00:00
categories: [blog]
tags:
  - angularjs
  - javascript
  - programming
---

#### Tree Widgets for Angular

If you google for angular tree widgets, you're likely to find advice that a
full-blown widget (or angular directive) is not necessary. If you need is
something that looks like Explorer/Finder then, of course, using an
off-the-shelf [treeview](https://github.com/eu81273/angular.treeview
"angular.treeview") or [control](https://github.com/wix/angular-tree-control
"angular-tree-control") is going to make life easier. But if all you need is to
represent your data in a tree structure then the language of angular directives
is expressive enough that representing a recursive data structure is simply a
matter of writing some recursive markup.
[This](http://jsfiddle.net/brendanowen/uXbn6/8/ "recursive fiddle") is possibly
the neatest example I've seen, but for the sake of completeness:

```html
<h2>A Tree</h2>
<div ng-include="'node.html'" ng-init="node=treeData"></div>
```

How can it be so simple? Why don't we need a new directive? I'm going to
explore these questions by comparing with more traditional desktop GUI toolkit
approaches to see where markup wins. Then I'm going to ignore this advice by
creating a directive anyway.

#### How would OO do it?

A traditional desktop UI toolkit would have taken an object-oriented approach
and identified an object with every visible element and then more objects to
manage the state. MVC is the poster child of OO design for UI code and this
tends to mean a suite of objects for the views (what you see on screen) and
another for the model (the data you want to represent). The remaining objects
can be input handlers in the real old school MVC, or more like glue logic in
the newer MV* family. OO found it's natural habitat in GUI toolkits because the
objects relate directly to something you can point to; often a region of the
screen. This makes it easy to reason about the code as it is so concrete, but
it can also be a little verbose. 

The difference in approach between markup and traditional OO is most marked
when you consider tables. If you aren't familiar with any desktop toolkits, try
[Java Swing's JTable](http://docs.oracle.com/javase/7/docs/api/javax/swing/JTable.html
"JTable Javadoc") for a fair representation. There are usually objects for the
table, the table model, the rows, the cells, headers, footers, columns; but
also column models and so on. Granted, most of these objects are hidden away
inside the toolkit, but if you want real control over the table, you will need
to deal with most of them. 

A tree widget in an OO toolkit is going to ask you to create an object to
represent the tree, supply the model as a recursive data structure and tailor
the presentation with parameters. Indeed, many angular widgets take the same
approach: one directive with a collection of attributes to parameterise the
display (templates etc.). But coming from the HTML direction, we already have a
sizeable toolkit for displaying nested lists. Tree widgets are a graphical
representation of tree data structures. Trees are simple recursive structures,
so it makes sense that the markup is too.

#### Is Markup Better?

A markup oriented toolkit is more of a language in that it gives you the basic
primitives from which to construct more complex entities. If you want some
fancy structure in your table rows then it's not a question of overriding the
default behaviour with parameters but of rearranging the language elements to
express the new structure. Angular adds to this feature of HTML with the
ability to define new basic language elements. This can be off-putting or it
can be very powerful: think lisp macros.

When it comes to constructing a fully featured tree view, it's not necessary to
create a tree widget that can include all the features. The elements can be
combined in exactly the same way as they would be for tables, lists etc. Let's
construct an example to make this clear: we want to add expand and collapse to
each level of the tree. Using the tutorial favourite
[zippy](http://www.thinkster.io/pick/JCDDOCEwVX/angularjs-building-zippy
"Egghead Tutorials"), it's trivial to modify the basic tree markup:

#### But...

I hope I've made a good enough case for building up angular views by combining
markup directives. In the specific case of tree views, it appears that no
specialist directive code is necessary. But I'm going to try wrapping up the
`ng-include` pattern above anyway. I am irrationally adverse to building up
from templates, but I would also like to make it more explicit that a section
of markup represents a tree. And sometimes it's just fun to make a directive.

So what do we need for a simple tree?

  1. The root element
  2. The content (template) for each element
  3. How to iterate over the child nodes
  4. How to nest the markup for child nodes

In the simple example at the top, we referred to the content template via
`ng-include` but also the child iterator in the same markup. We hooked up to
the root element via the `ng-init` reference to `treeData`.

Instead, we could design something that looks like a repeater (you might have
noticed my fondness for them) and as long as we have those 4 bits of
information, we should be able to re-create the tree.

```html
<ul>
  <li sf-treepeat="node in children of treeData">
    {% raw %}{{node.name}}{% endraw %}
    <ul>
      <li sf-treecurse></li>
    </ul>
  </li>
</ul>
```

Which should compile as below.

```html
<ul>
  <li sf-treepeat="node in children of treeData">
    {% raw %}{{treeData.name}}{% endraw %}
    <ul sf-treecurse>
      <li ng-repeat="node in treeDatachildren">
        {% raw %}{{node.name}}{% endraw %}
        <ul sf-treecurse></ul>
      </li>
    </ul>
  </li>
</ul>
```

#### Coding angular-tree-repeat

To make a tree repeater, we need 2 directives: `sf-treepeat` and `sf-treecurse`
(I'm sorry, I couldn't help myself with the names) and the inner will replace
itself with the content of the outer for each child node. This replacement is a
link-time operation (because it depends on the model) using information
captured at compile time by the outer directive. A controller handles the
communication.

To start, lets create a module:

```javascript
var mod = angular.module('sf.treeRepeat');
```

and a parser for the repeat expression (this should be familiar from
[before](http://blog.stackfull.com/2013/03/angularjs-virtual-scrolling-part-2/
"AngularJS Virtual Scrolling – part 2"))

```javascript
function parseRepeatExpression(expression){
  var match = expression.match(/^\s*([\$\w]+)\s+in\s+([\S\s]*)\s+of\s+([\S\s]*)$/);
  if (! match) throw new Error("...");
  return {
    value: match[1],
    collection: match[2],
    root: match[3]
  };
}
```

Now for the outer directive `sf-treepeat`. This has 2 jobs: create a controller
that makes the node available in scope and grab the HTML content of the element
before it has been compiled. This content is stored on the controller.

```javascript
mod.directive('sfTreepeat', function() {
  return {
    restrict: 'A',
    scope: true,
    controller: function ($scope, $attrs){
      var ident = this.ident = parseRepeatExpression($attrs.sfTreepeat);
      $scope.$watch(this.ident.root, function(v){
        $scope[ident.value] = v;
      });
    },
    compile: function (element){
      var template = element.html();
      return {
        pre: function (scope, element, attrs, controller){
          controller.template = template;
        }
      };
    }
  };
});
```

Finally, the inner directive which simply replaces itself. The controller is
pulled in via the `require`. The new element must be `$compile`'d and linked to
the scope.

```javascript
mod.directive('sfTreecurse', function($compile){
  return {
    require: "^sfTreepeat",
    link: function (scope, iterStartElement, attrs, controller) {
      var build = [
        '&lt;', iterStartElement.context.tagName, ' ng-repeat="',
        controller.ident.value, ' in ',
        controller.ident.value, '.', controller.ident.collection,
        '">',
        controller.template,
        '&lt;/', iterStartElement.context.tagName, '>'];
      var el = angular.element(build.join(''));
      iterStartElement.replaceWith(el);
      $compile(el)(scope);
    }
  };
});
```

#### Done

The directive code is undoubtedly more complex and difficult to read than the
original markup and the markup using the new directives is only marginally more
readable than the original. But it's an indication of what can be done. I have
to confess that I originally wanted to make it work with the transclusion
functions rather than HTML pulled from the outer element. I spent a few
frustrating hours trying different combinations &ndash; some tantalisingly
close to working &ndash; before I had to call time and stick with what works.

The real code (with demos) is [up on
github](https://github.com/stackfull/angular-tree-repeat
"angular-tree-repeat"). If you find a way to work it without html chopping,
please let me know!

