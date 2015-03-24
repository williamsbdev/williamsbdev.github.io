---
layout: post
title: "Extending Ember with ember-cli addons"
tags: [ember, ember-cli]
---

I am often concerned with the many design patterns that are considered best
practices. The [single responsibility
principle](http://en.wikipedia.org/wiki/Single_responsibility_principle) is one
that can easily be violated. As I work in Ember, I find that when writing my
tests, it can be easy to just keep the test passing quickly by putting all my
code in the controller. This a clear violation of the single responsibilty
principle. The focus of this post will be to extract the data persistence layer
out and show-off some of the addons to make this easy to do so.

What my team has found was that my data persistence layer wants to leak into
the controller.  A first attempt solution was to put the persistence logic into
the model. This tended to bloat the model, distracting from what the model
really was and also violated the the single responsibility principle. The
second attempt was to create a repository object that was responsible for
persisting the objects or fetching the objects from the server.  This pattern
seemed to be working really well as the model would know how to validate it's
own fields and then the repository was concerned with the saving of the object
on the server. Each repository object had it's own reponsibility. However, as
our number of models grew, so did the number of our repositories. We were
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

An idea was a way to programmatically register all the objects (repositories)
into the application. Since all the repositories were in a single directory
(the `repositories` directory) we needed to find a way to register all the
objects in the directory just like ember-cli does. What we came up with was
[ember-cli-auto-register](https://github.com/williamsbdev/ember-cli-auto-register).
This helped the situation by cutting out a couple of lines (the
`application.register` lines) but we still had the problem of all the
injections.

What we really wanted was a way to inject the objects we had registered with
the application into the routes or controllers directly. We did not want to go
to an initializer to do so. Ember came out with the `Ember.inject.service` and
`Ember.inject.controller` API in 1.10 but this was limited only to objects of
type `Ember.Service` and `Ember.Controller`. Since we had created our own
repository objects and registered them of type `repositories`, we needed a way
to inject them. So was born the ember-cli addon
[ember-cli-injection](https://github.com/williamsbdev/ember-cli-injection).
With the combination of these two addons, we were able to reduce our
initializers down to a single file to register all the objects. See the
improved code below:

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
`repositories` directory and will be registered with the name of the file as
the name of the object. So for example, the `import PersonRepository from
"app/repositories/person";` line that we were able to delete, would be
registered the exact same way as we had done manually
`application.register("repositories:person", PersonRepository);`. The main
advantage that we saw to this was that as we added another repository, it was
automatically registered into the application. Once we setup the initializer,
we could walk away and never worry about a repository not being available for
us to inject.

Now the injection piece is setup by creating function that will lookup the
object in the container with the specified type of object. We have started by
creating a utilities function:

```javascript
// app/utils/inject.js
import injection from "ember-cli-injection/inject";

export default injection("repositories");
```

Once you have created this injection function, you can simply inject the
desired repository into any route or controller with the following:

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

This more explicit approach is something that Ember is trending towards with
the `Ember.inject` API. It puts the variable declaration right in the class
where it is used, which will also allow you to quickly see if you have a
property being injected that is no longer used.
