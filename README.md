# Drupal 8 JavaScript Path Forward

This is a proposed path forward for JavaScript improvements in Drupal 8.

**_Note, this document is a draft. It should be treated as such with improvements, alterations, and suggestions._**

_This was written because both JohnAlbin and Jacine before him asked for it._

## Assumptions and Guidelines
Before we can list out tasks to address we need to look at the assumptions and guidelines surrounding Drupal and JavaScript. How and what is implemented should be based on a set of assumptions and an understanding of the audience this affects.

* __Drupal developers are not JavaScript developers.__ In the Drupal community, many of the developers who write JavaScript are not JavaScript developers. We need to improve the JS code in core so that it is an exemplar of good JavaScript code and practices. We want to raise the quality of Drupal's JS while not making that quality unachievable to non-JS developers.
* __Faster page performance matters to users.__ Metrics are regularly showing that the use of JavaScript in pages are increasing, that this is slowing down page performance, and that page performance matters to end users. They expect everything to be fast and we need to aide this.
* __Data and technical explanations over opinions.__ When it comes to JavaScript there are a lot of opinions, some of them contradict each other, and some of them are on technical topics that are out of date or no longer relevant. We need to take a technical look at what we are going to do and why we are going to do it in that way.
* __The window to work on Drupal 8 is limited.__ Drupal 8 new features have a limited window of time to be written, tested, and implemented. We need to look at what useful elements we can get in during the development time remaining.

## The Path Forward
This is a path forward for Drupal 8 with regard to JavaScript. This is not a comprehensive path forward for Drupal but a hopefully attainable one for Drupal 8.

### Minify all core JavaScript.
To see the benefit take the simple example of drupal.js. The minified version (with UglifyJS - the tool jQuery currently uses) is 24% the size of the original. Similar improvements can be seen with other core JavaScript files. With no functionality loss we save on the amount of data shipped to each browser. For some real world data see the article "[Why Minify JavaScript?](http://engineeredweb.com/blog/why-minify-javascript/)" or the [presentation on front end performance from DrupalCon Denver](https://speakerdeck.com/u/mattfarina/p/front-end-performance-improvements?slide=24).

As an added bonus, if broken JavaScript is passed into UglifyJS it will return an error and help us identify bugs. The first few releases of Drupal 7 had broken JavaScript in it.

### Pluggable asset (JS/CSS) handling.
There is no one solution for how to handle JS and CSS bundling. The use case of a blog on shared hosting or a social community site on a scaled out network are very different in how users interact. Core should ship with a sane middle of the road setup while allowing contrib authors and site owners to replace this system with one better suited for their use case.

### Offload asset (JS/CSS) generation from the current page request.
This is similar to imagecache. The main page request can, at times, take well over a second when it has to generate new aggregate JS and CSS files. This main page request has a finite amount of memory as well. But moving JS and CSS aggregation and preprocessing to separate we can free up memory in the main page request and move this work into parallel processes. Overall this will cause more resource usage (a Drupal bootstrap for each resource rather than one for the page) but will free up more resources per process.

### Default JavaScript inclusion to the page footer.
The best practice for JavaScript is to include JavaScript in the footer unless you have a reason to include it later. Essentially, include JavaScript files as late as possible rather than as early as possible. Core currently does the latter.

### User the module pattern properly with full dependency injection.
Our current JS usage is a mild modification of the jQuery plugin pattern with globals. For example,

```
(function($){
  Drupal.foo = {};
})(jQuery)
```

This is properly using dependency injection to pass in jQuery but we still handle Drupal as a global. Instead we should use something in the form:

```
(function($, Drupal){
  Drupal.foo = {};
})(jQuery, Drupal)
```

If we use other globals (e.g., document) we should pass those in as well. For example,

```
(function($, Drupal, document){
  Drupal.foo = {};
})(jQuery, Drupal, this.document)
```

Not only does this deal with dependency injection properly, tools that minify JS can make them even smaller. By passing Drupal in as a dependency in drupal.js the size could be minified to 22.5% the original size. That is an extra 1.5% savings.

For more information on the module pattern see the article "[JavaScript Module Pattern: In-Depth](http://www.adequatelygood.com/2010/3/JavaScript-Module-Pattern-In-Depth)".

### Fix errors reported by JSHint
Run the Drupal JavaScript through [JSHint](http://www.jshint.com/) and fix the errors. This will improve code quality.

### Add Named Groups for Assets To The API
Drupal core currently has 3 groups in JS_LIBRARY, JS_DEFAULT, JS_THEME. Instead we should move toward named groups for aggregation/description. For example, if there is a group for WYSIWYG this could be aggregated separately. Then that aggregate included on just pages where WYSIWYG is needed. The browser can do a better job caching aggregates this way.

_Note, this should be part of the description when JavaScript is added. Whether an aggregator uses this or not is up to the aggregator._

For more information on loaders see the [Pagespeed presentation from the Velocity conference](http://pagespeed-velocity2011.appspot.com/).

### Modernizr + YepNope
In truth I'm not sure if this belongs in core or contrib. But, it is very useful. This idea is simple. Detect what features your browser has and then optionally load some scripts based on this. A setup like this is no switching to full on script loading (see below for more on that) but instead about gracefully handing browser/version features which has grown into an issue.

## What Is Not Currently In The Plan
Some things are not currently in the plan but could be. They are not in the plan because of a few reasons:

1. There isn't enough time to get them in along with solving some more immediately needed changes.
2. A decrease in DX for a core developer audience without a mitigation plan.
3. We still need more data/justification for a change.

### Replace Drupal.theme() with template system
Since Drupal.theme came to life the JS communities have really jumped into dealing with the templating problem with tools like [Mustache](http://mustache.github.com/) and [Handlebars](http://handlebarsjs.com/). We should replace the core Drupal system with a 3rd party system that rocks. This task is not included in the plan in the interest of time which there is not a lot of and I have not seen someone champion this in core.

### Using A Full JavaScript Loader (e.g. require.js)
Using loaders is rather popular in the JavaScript community. When you are building the entire UI in JavaScript and want to call back for any functionality loading that on the fly is great. But, Drupal doesn't generate pages this way, there is debate around if this is a good idea, not all the JavaScript devs are for it, there are DX issues for contrib, and this is not a small task. These are all problems that should be solved before moving forward. Let me go into detail on some of problems facing this:

1. A lot of Drupalers are not hard core JavaScript developers. They know just enough jQuery to be dangerous and get things built for their needs or their clients. We need to keep things as simple as possible for this group. Developers in this group regularly drop an external script into Drupal and call it based on some example on that scripts project page. _How do we keep it simple for this group of devs?_
2. Part of the reason for script loaders is performance. But, [according to the Google Pagespeed team we shouldn't use a loader for core scripts needed for the paint](http://pagespeed-velocity2011.appspot.com/#17). That means we need to segment what should be loaded by a script loader and what's not loaded. This would be different for different pages. This is a rather complex problem.
3. Making this change will take time. There is a small window to get some long overdue low barrier to entry changes in.

*Note, script dependency handling already happens through hook_library(). This may not be optimal or some form of standard. But, it works, developers with Drupal 7 are familiar with it, and until we have a better approach that deals with all the issues it does provide a mechanism for dependency handling.*

## Todo

1. Link in the relevant issues (if any exist) or start issues.
2. Go have a milkshake.