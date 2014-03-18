---
layout: post
title: "Open Source"
category: oss
tagline: 'My first Contribution'
tags: [oss, coffee, grunt]
---

I have been developing web applications for nearly four years and it was just
four months ago that I made my first open source contribution. I suffered from
imposter syndrome like so many developers do (great podcast by Ruby Rogues on the
[imposter syndrome](http://rubyrogues.com/107-rr-impostor-syndrome-with-tim-chevalier/)
issue).

We were using the build tool [Grunt](http://gruntjs.com/) at work to
compile our front-end code. I am a huge [CoffeeScript](http://coffeescript.org/)
fanboy so I wanted to be able to write our Gruntfile in CoffeeScript. I could write
my Gruntfile in CoffeeScript however our Gruntfiles were getting large and to solve
that problem, my coworker told me about modularizing your Gruntfile with the
[load-grunt-config](https://github.com/firstandthird/load-grunt-config) module.
As I was playing with this module at home I noticed that it only supported js
and yaml files at the time. I thought, "Why don't they support CoffeeScript? I
guess I will open an issue.".  Before I did that I started looking at the
project and diving into the source of this minimal project, and I realized I
could figure it out. I saw they had some tests, so I wrote a test, added the
code and a sample file. Here is my
[commit](https://github.com/williamsbdev/load-grunt-config/commit/b683807aa9cccf0d54f4336fcf185e47f9aac567).

As you can see, I did almost nothing for this project except make myself and
all subsequent CoffeeScript enthusiasts happy. All I am trying to say is that
you can contribute to open source. It does not need to be a complete fix of the
issue on a project that has been going on for three years with 117 posts on the
thread. It can be as simple as starting out with a spelling error in a README.
Fork the project, fix the spelling mistake and submit a pull request.
[Github](http://github.com) makes it super easy and once you submit your first
pull request, the world Open Source Software will not be so big and scary.
