---
layout: post
category : ember
tagline: "A first impression"
tags : [ember, promises, coffeescript]
---

I am working on an [Ember](emberjs.com) application outside of work to gain a
better understanding of the framework. I am also using [Ember
Data](https://github.com/emberjs/data) and it is really cool in the way that it
uses promises. I ran into a scenario where I was dealing with one of these
promises, and as is usual practice for me, I was making things more complicated
than they needed to be. Here is my thought process as I encountered my first
Ember promise.

I have the following models in my application:

{% highlight coffeescript %}
App.Shift = DS.Model.extend
  name: DS.attr 'string'
  people: DS.hasMany 'person'

App.Person = DS.Model.extend
  first_name: DS.attr 'string'
  last_name: DS.attr 'string'
{% endhighlight %}

In my Handlebars template I wanted to display the number of people for a given
shift:

{% highlight HTML %}
  {{ "{{shift.name"}}}} - {{ "{{shift.number_of_people"}}}}
{% endhighlight %}

On my shift I thought I would add the computed property of number_of_people.
This would take the array of people and return the length of the array (as some
more experienced ember people will notice, I'm doing a lot of things wrong but
we will fix it by the end).

{% highlight coffeescript %}
App.Shift = DS.Model.extend
  name: DS.attr 'string'
  people: DS.hasMany 'person'
  number_of_people: (->
    @get('people').length
  ).property('people')
{% endhighlight %}

Seemed fairly straight-forward to me, however, I encountered this error in the
console:


{% highlight javascript %}
  Assertion failed: You looked up the 'people' relationship on
  '<App.Shift:ember277:1>' but some of the associated records were not
  loaded. Either make sure they are all loaded together with the parent record,
  or specify that the relationship is async ('DS.hasMany({ async: true })')
{% endhighlight %}

This was awesome. My first error in Ember and it gave me a helpful error
message telling me exactly what I need to do to fix it. So I did the following:

{% highlight coffeescript %}
App.Shift = DS.Model.extend
  name: DS.attr 'string'
  people: DS.hasMany 'person', async: true
  number_of_people: (->
    @get('people').length
  ).property('people')
{% endhighlight %}

This fixed my error but now I was not seeing anything in my template. I was
curious what the people looked like so I just returned the people from my
computed property.

{% highlight coffeescript %}
App.Shift = DS.Model.extend
  name: DS.attr 'string'
  people: DS.hasMany 'person', async: true
  number_of_people: (->
    @get('people')
  ).property('people')
{% endhighlight %}

Ah. It is an Ember PromiseArray! After learning some more about promises, I
figured out I needed to do a .then with a function and my people passed to that
function.

{% highlight coffeescript %}
App.Shift = DS.Model.extend
  name: DS.attr 'string'
  people: DS.hasMany 'person', async: true
  number_of_people: (->
    @get('people').then((people)->
      people.length
    )
  ).property('people')
{% endhighlight %}

Still not what I was looking for. Now I was getting an object in my template
and it was being displayed:

{% highlight HTML %}
  [object Object]
{% endhighlight %}

I was confused on how to return the length of my people array once the promise
has been resolved from my computed property (remember how I said I make things
more complicated than they need to be). I threw a bunch of spaghetti at the
wall and none of it was working. I was stuck. So I threw a question on
[StackOverflow](http://www.stackoverflow.com). The person answering asked me
why I was trying to get the length of the array in a computed property. Instead
to try this:

{% highlight HTML %}
  {{ "{{shift.name"}}}} - {{ "{{shift.people.length"}}}}
{% endhighlight %}

In the Handlebars, Ember knows that the people property is a promise and will
evaluate the promise and then try to do the .length after the promise has been
resolved. So my model went back to being super simple.

{% highlight coffeescript %}
App.Shift = DS.Model.extend
  name: DS.attr 'string'
  people: DS.hasMany 'person', async: true
{% endhighlight %}

What did I learn, if it seemed like I was working too hard in Ember, I probably
am. While I'm not really taking advantage of the promise for people (I'm not
grabbing any values off of the people being returned), it was still cool to see
how promises work from templates. My first impression of Ember and it's
promises, positive.
