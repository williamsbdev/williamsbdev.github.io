---
layout: post
title: "Ember url params from parent route"
tags: [ember]
---

Today while I was working with Ember, I discovered that a child route, which
had a dynamic url, needed access to the parent routes dynamic url segment. I
had tried to do this once before without success. However, today I thought I
would try this again and found this awesome [StackOverflow question]. The
solution I found most interesting was the answer from [codefox421]. His
solution is below and allowed me access to the parent object id from the url
which is exactly what I wanted.

```javascript
// app/router.js
App.Router.map(function () {
  this.route('dimensions', function() {
    this.route('dimension', { path: ':dimension_id'}, function () {
      this.route('value', { path: 'values/:value_id' });
    });
  });
});
```

```javascript
// app/routes/dimensions/dimension/value.js
export default Ember.Route.extend({
  model: function(params, transition) {
    var dimension_id = transition.params.dimension.dimension_id;
    // do whatever you need with dimension_id and params.value_id
  }
});
```


[StackOverflow question]: http://stackoverflow.com/questions/22376994/ember-deeply-nested-routes-do-not-keep-parent-dynamic-parameter
[codefox421]: http://stackoverflow.com/users/2085526/codefox421
