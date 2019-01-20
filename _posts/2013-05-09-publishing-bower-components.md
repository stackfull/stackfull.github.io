---
layout: post
title: Publishing Bower Components
date: 2013-05-09T20:39:43+00:00
categories: [blog]
tags:
  - modules
  - javascript
  - programming
---
#### Bower

[Bower](http://bower.io/) is a light-weight package manager for web application
components. Packages are hosted as git repositories and registered with the
bower registry. Being so lightweight, it's a simple matter to publish a
component that other people can
[find](http://sindresorhus.com/bower-components/) and include in their apps. 

```shell
$ bower init
  ...answer some questions...
$ git tag v1.0.0
$ git push origin -a
$ bower register my-component-name git@my.server:path/to/repo.git
```

There is, naturally, a downside to this simplicity. It's very tempting to
register your source repository as a component and many people do this. But
most components will need building before they are useful (minification,
transpiling, LESS/SASS etc.) and this leads to the dilema: should you check in
built artifacts to your source repo? 

Like all such issues, it's a matter of personal taste. JS is unusual in that
built and source are really the same kind of file, so it makes a sort of sense.
This means it happens a lot. But I find it confusing when components are pulled
into a build complete with all source, documentation, tests and so on.

The alternative is maintaining 2 repositories for a component. This can annoy
people, but most other development environments have a clear separation between
the code and the published product. So I'd like to outline a recipe for taking
a little of the pain out of it.

#### The Other Repo

The approach I took when publishing [the virtual scrolling
component](https://github.com/stackfull/angular-virtual-scroll) was to have a
separate repository for the bower component and use git submodules to map that
as the `dist` directory of the source repository. The build (managed by
[Grunt](http://gruntjs.com/)) puts the minified files in the `dist` directory
and anyone cloning the source repo won't notice any difference.

To set this up, use the `git submodule add` command as described
[here](http://git-scm.com/book/en/Git-Tools-Submodules). The component
repository is just a regular git repository with nothing in it (for now). It
must be set up first, because the submodule link will point to the specific
version. Then in the source repository:

```shell
$ git submodule add <component-repo-url> dist
```

Submodules aren't activated by default, so any clone of the source repo will
behave as if `dist` is just a normal directory. To really use the 2
repositories together requires the following sequence of actions:

1. Update the submodule to the latest:
```shell
$ git submodule init
$ git submodule update
$ (cd dist; git checkout master)
```

1. Build (be sure you are building a release with new version numbers etc.). 

1. Write new version info to the component description file
   (`dist/component.json` or `dist/bower.json`). 

1. Commit and tag in the submodule. 
```shell
$ (cd dist; git commit -a -m"New version")
$ (cd dist; git tag -m"New version")
```

1. Commit and tag in the source repo.
```shell
$ git commit -a -m"New version"
$ git tag -m"New version"
```

1. Update the submodule again. 

1. Push both repositories to the public endpoints.


Some of these steps are easily overlooked, so I have them coded as grunt tasks
(see the
[Gruntfile.js](https://github.com/stackfull/angular-virtual-scroll/blob/master/Gruntfile.js)).
Feel free to copy and modify whatever will help your builds.

