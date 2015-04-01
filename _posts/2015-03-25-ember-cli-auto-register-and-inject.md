---
layout: post
title: "Extending Ember with ember-cli addons"
tags: [ember, ember-cli]
---

I am often concerned with the many design patterns that are considered best
practices. As my team was developing our current set of applications, they
start out like all apps do, small. As it grew we began to feel pain around the
objects that were not part of the [Ember.js] ecosystem.

What my team has found was that as we added singleton objects, repositories,
that we wanted to use throughout our applications, we would create an
initializer that was responsible for registering our new repository and
injecting it where needed. So every time we created a new repository, we
created another initializer. This was less than desirable. What we ended up
with was something like the below code example several times over.

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

We really wanted to programmatically register all the repositories into the
application. Since all the repositories were in a single directory (the
`repositories` directory) we needed to find a way to register all the objects
in the directory just like [ember-cli] does for some of the other first class
objects in the Ember.js ecosystem. What we came up with was
[ember-cli-auto-register] which took the idea from the
[ember-load-intializers]. This helped the situation by cutting out a couple of
lines (the `application.register` lines) but we still had the problem of all
the injections.

What we really wanted was a way to inject the objects we had registered with
the application into the routes or controllers, within the route or controller
itself. We did not want to go to an initializer to do so. Ember came out with
the `Ember.inject.service` and `Ember.inject.controller` API in 1.10 but this
was limited only to objects of type `Ember.Service` and `Ember.Controller`.

Since we had created our own repository objects and registered them of type
`repositories`, we thought there could be a way to inject them similar to how
the `Ember.inject` API worked. So was born the ember-cli addon
[ember-cli-injection]. This addon will allow you create a function that will
lookup an object based on the type that you use to initialize the function.

```javascript
import injection from "ember-cli-injection/inject";

export default injectRepository("repositories");
```

With the combination of these two addons, we were able to reduce our
initializers down to a single file to register all the objects.  See the
improved code below:

```javascript
// app/initializers/repositories.js
import register from "ember-cli-auto-register/register";

export function initialize(container, application) {
    register("repositories", application);
    // every repository will use this 'store' object so just like other
    // objects we can push some object into all objects of type 'repositories'
    application.inject("repositories", "store", "store:main");
}

export default {
    name: "repositories",
    after: "store",
    initialize: initialize
};
```

This single initializer will register every repository object in the
`repositories` directory and will be registered with the name of the file, as
the name of the object in the application. So for example, the
`PersonRepository` from "app/repositories/person", would be registered the
exact same way as we had done manually
`application.register("repositories:person", PersonRepository);`. The main
advantage was that as we added another repository, it was automatically
registered into the application. Once we setup the initializer, we could walk
away and know every new repository was available for us to inject.

Now the injection piece is setup by creating a function that will lookup the
object in the container with the specified type of object (as we have seen above
already). We started by creating a utilities function:

```javascript
// app/utils/inject.js
import injection from "ember-cli-injection/inject";

export default injectRepository("repositories");
```

Once you have created this injection function, you can inject the desired
repository into any route, controller, other object with the following:

```javascript
// app/routes/people.js
import Ember from "ember";
import injectRepository from "app/utils/inject";

var PeopleRoute = Ember.Route.extend({
    repository: injectRepository("person"),
    model: function() {
        return this.get("repository").fetch();
    }
});

export default PeopleRoute;
```

This more explicit approach is something that, I believe, Ember is trending
towards with the `Ember.inject` API. It puts the variable declaration right in
the class where it is used, which will allow you to quickly see to which
objects you have access. It also points out objects being injected that are no
longer used. The maintenance cost of initializers is now minimal. Also, having
the injected properties declared in the controller allows us to get more work
done. We do not have to change files (back to the initializers) to figure if
the object is injected, or inject the object if it is not already injected. The
tighter feedback loop is our ultimate goal when building and testing
applications.

[Ember.js]: http://emberjs.com
[ember-cli]: http://ember-cli.com
[ember-cli-injection]: https://github.com/williamsbdev/ember-cli-injection
[ember-cli-auto-register]: https://github.com/williamsbdev/ember-cli-auto-register
[ember-load-intializers]: https://github.com/ember-cli/ember-load-initializers
