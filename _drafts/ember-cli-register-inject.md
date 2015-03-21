---
layout: post
title: "Extending Ember with ember-cli addons"
tags: [ember, ember-cli, javascript]
---

I am often concerned with the many design patterns that are considered best
practices. The [single responsibility
principle](http://en.wikipedia.org/wiki/Single_responsibility_principle) is one
that can easily be violated. As I work in Ember, I find that when writing my
tests, it can be easy to just keep the test passing quickly by putting all my
code in the controller. This a clear violation of the single responsibilty
principle.

A pattern I have found is that, my data persistence layer wants to leak into
the controller. A first attempt solution was to put the persistence logic onto
the model. This tended to bloat the model and distract from what the model
really was. The second attempt was to create a repository object that was
responsible for persisting the objects or fetching the objects from the server.
This pattern seemed to be working really well as the model would know how to
validate it's own fields and then the repository was concerned with the saving
of the object on the server. Each object had it's own reponsibility. However,
as our number of models grew, so did the number of our controllers. We were
registering the repositories and injecting them into the routes or controllers
that needed them. What we ended up with was something like this:

```javascript
// app/initializers/person-repository.js
import PersonRepository from "app/repositories/person";

export function initialize(container, application) {
    application.register("repositories:person", PersonRepository);
    application.inject("repositories:person", "store", "store:main");
    application.inject("route:people", "repository", "repositories:person");
    application.inject("route:people/person", "repository", "repositories:person");
    application.inject("controller:people/person", "repository", "repositories:person");
    application.inject("controller:add", "repository", "repositories:person");
    application.inject("route:add", "repository", "repositories:person");
}

export default {
    name: "person-repository",
    after: "store",
    initialize: initialize
};
```

For reach model, we had a corresponding repository that knew how that model was
to be translated back to the server. As you add more and more models and
repositories, your initializers become unmanageable. I would create a new
initializer to register that repository and make sure it was injected into the
correct places.

A solution that I came up with was the
[ember-cli-auto-register](https://github.com/williamsbdev/ember-cli-auto-register).
This helped the situation by cutting out a couple of lines but I still had the
problem of all the injections.

What I really wanted was a way to inject the objects I had registered with the
application into the routes or controllers directly. I did not want to go to an
initializer to do so. Ember came out with the `Ember.inject.service` and
`Ember.inject.controller` API in 1.10 but this was limited only to objects of
type `Ember.Service` and `Ember.Controller`. I had created my own repository
objects of type "repositories". So was born the ember-cli addon
[ember-cli-injection](https://github.com/williamsbdev/ember-cli-injection).
With the combination of these two addons, I was able to reduce my initializers
down to a single file to register all the objects. See the improved code below:

```javascript
// app/initializers/repositories.js
import register from "ember-cli-auto-register/register";

export function initialize(container, application) {
    register("repositories", application);
    application.inject("repositories", "store", "store:main");
}

export default {
    name: "repositories",
    after: "store",
    initialize: initialize
};
```

This single initializer will register every repository object in the
`repositories` directory and will be register as the name of the file, so for
example, the `import PersonRepository from "app/repositories/person";` line
that I was able to delete, would be register the exact same way as I had done
manually `application.register("repositories:person", PersonRepository);`.  The
main advantage that I saw to this was that as I added another repository, it
was automatically registered into the application. Once I setup the
initializer, I could walk away and never worry about a repository not being
available for me to inject.

Now the injection piece is setup by creating a simple function.

```javascript
// app/utils/inject.js
import injection from "ember-cli-injection/inject";

export default injection("repositories");
```

Once you have created this injection function, you can simply inject the
repository into any desired route or controller with the following:

```javascript
// app/routes/people.js
import Ember from "ember";
import inject from "app/utils/inject";

var PeopleRoute = Ember.Route.extend({
    repository: inject("person"),
    model: function() {
        return this.get("repository").fetch();
    }
});

export default PeopleRoute;
```

This approach is more explicit as you can see in your route or controller
everything that is injected. It will also allow you to quickly see if you have
a property being injected that is no longer used.
