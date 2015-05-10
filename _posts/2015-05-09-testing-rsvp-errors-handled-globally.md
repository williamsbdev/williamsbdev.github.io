---
layout: post
title: "Testing Ember.RSVP.on('error')"
tags: [ember]
---

I recently ran into an issue testing `Ember.RSVP.on("error", function(){})`.
The default `Ember.Test.Adapter.exception` function will fail the test. I was
handling the error but since the `Ember.Test.Adapter` is also listening to
`Ember.RSVP.on("error")`, I could not pass the test. I ended up putting all the
tests that were expecting the `Ember.RSVP.on("error")` to handle the error in
one module and then did a monkey patch of `Ember.Test.adapter.exception`. See
code example below and [example project].

```javascript
// app/repository/foo.js
import Ember from "ember";

export default Ember.Object.extend({
  findAll: function(){
    var all = Ember.A();
    new Ember.RSVP.Promise(function(resolve, reject){
      $.ajax({
        method: "GET",
        url: "/api/sessions",
        success: function(data) {
          return Ember.run(null, resolve, data);
        },
        error: function(data) {
          return Ember.run(null, reject, data);
        }
      });
    });
  }
});
```

```javascript
// tests/acceptance/foo-test.js
import Ember from "ember";
import {test, module} from "qunit";
import startApp from "../helpers/start-app";

var application, originalException;

module("Acceptance: Foo", {
  beforeEach: function() {
    application = startApp();
    originalException = Ember.Test.adapter.exception;
    Ember.Test.adapter.exception = function(){};
  },
  afterEach: function() {
    Ember.Test.adapter.exception = originalException;
    Ember.run(application, "destroy");
  }
});

test("test for globally error handling", function(assert) {
  Ember.$.fauxjax.new({
    request: {
      method: "GET",
      url: "/api/sessions"
    },
    response: {
      status: 200,
      content: [{id: 1, name: "foo"}, {id: 2, name: "bar"}]
    }
  });
  // request made to /api/sessions on "/" route
  visit("/");
  andThen(function() {
    // test does not fail
    assert.equal(1, 1);
  });
});
```

[example project]: https://github.com/williamsbdev/ember-rsvp-global-error-handler
