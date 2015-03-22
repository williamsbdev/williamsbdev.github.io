---
layout: post
title: "Ember Productivity"
category: ember
tagline: "get things done"
tags: [ember]
---

It was a year ago that we first introduced [Ember.js](http://emberjs.com) into
our stack. Now we have three products built or being built on the technology.
There are many other JavaScript libraries and frameworks out there but I am
thankful to be on a team that has chosen Ember for the front-end of our
stack.

I liken the decision to use Ember to our choice for our backend framework.
We use the Python web frameworks Django and Django REST. I find it really nice
that I can jump into any of our Django applications and quickly figure out what
is going on because of the strong conventions and opinions that Django has.
Some people may not like having the opinionated framework or language like
Django and Python, respectively. However, I believe that it improves my
productivity as I can spend only a few minutes getting up to speed on the
information the application is providing to users and what it is trying to
accomplish.

I find that this story is quite the same for Ember. With the strong conventions
established for naming a route, template, controller, and view, it is easy to
find the different needed pieces of the application. When I come into a project
that is new to me, I will start with the router. As Tom and Yehuda said during
their keynote at the Fluent conference, "The Router is the Heart and Soul of an
Application". After getting my bearings with the name and flow of the routes, I
can find any other piece of the application quickly. Below is a quick
CoffeeScript example of an Ember Router:

```coffeescript
  App.Router.map ->
    @resource 'users', {path: '/users/'}
    @resource 'shirt', {path: '/shirts/:size/'}
```

Given the above example, I now want to know what information will be available
to me when I navigate to the users route. Knowing only the name of the route
"users", I will be able to quickly find the UsersRoute. See example below:

```coffeescript
  App.UsersRoute = Ember.Route.extend
    model: ->
      App.Users.find()

  App.ShirtRoute = Ember.Route.extend
    model: (params) ->
      App.Shirt.find(params.size)
```

Now that I know that the data that will be available for each of the routes, I
can go to the users.handlebars or shirt.handlebars templates and see how the
information is being displayed for the user.

If there is additional functionality for those views into the data, I might be
able to find the UsersController or the ShirtController. If there was nothing
special going on with the views into the data, then Ember would provide the
generic UsersController and ShirtController.

```coffeescript
  App.UsersController = Ember.ArrayController.extend()

  App.ShirtController = Ember.ObjectController.extend()
```

With all these naming conventions, it is easy on me to come into a project
another team member has built and understand quickly what is happening in the
application. The one downside I see to having frameworks like Django and Ember
is learning the conventions that the framework has in place.  This is when
great documentation is an absolute must have.

Overall, I love having the conventions as all the trivial decisions of what to
name the different objects has already been decided. Frameworks, also help
provide a common structure for projects so that others can jump into an
application and be productive with only a short tour of the code.
