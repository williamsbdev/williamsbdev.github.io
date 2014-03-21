---
layout: post
title: "Ember Productivity"
category: ember
tagline: "get things done"
tags: [ember, coffeescript, python, django]
---

It was a year ago that we first introduced Ember.js into our stack. Now we have
three products built or being built on the technology. There are many other
JavaScript frameworks out there but I am thankful to be on a team that has
foresight.

I liken the decision to use Ember.js to our choice for our backend framework.
We use the Python web frameworks Django and Django REST. I find it really nice
that I can jump into any of our Django applications and quickly figure out what
is going on because of the strong conventions and opinions that Django has.
Some people may not like having the opinionated framework or language like
Django and Python, respectively. However, I believe that it improves my
productivity as I can spend only a few minutes getting up to speed on the
information the application is providing to users and what it is trying to
accomplish.

I find that this story is quite the same for Ember.js. With the strong
conventions established for naming a route, template, controller, and view, it
is easy to find the different needed pieces of the application. When I jump
into an application, I start with the router which is the core of the
framework.  Everything pivots around this seemingly small object. Based on the
name of the resource/route in the router, you will find a corresponding route
object. For example:

```coffeescript
  App.Router.map ->
    @resource 'users', {path: '/users/'}
    @resource 'shirt', {path: '/shirts/:size/'}

  App.UsersRoute = Ember.Route.extend
    model: ->
      App.Users.find()

  App.ShirtRoute = Ember.Route.extend
    model: (params) ->
      App.Shirt.find(params.size)
```

The above routes would have corresponding handlebar templates users.handlebars
and shirt.handlebars. I would be able to count on there being controllers named
according App.UsersController and App.ShirtController. This makes it quite easy
to jump into anyone's code and quickly find all the necessary pieces to begin
putting the puzzle together. The one downside I see to having frameworks like
Django and Ember is learning the conventions that the framework has in place.
This is when great documentation is an absolute must have. Overall, I love
having the conventions as all the trivial decisions of what to name the
different objects has already been decided and it also cuts down on team
discussion as to what it should be called. You can also talk with other
developers on completely different teams and know exactly what they are talking
about since you have a common terminology.
