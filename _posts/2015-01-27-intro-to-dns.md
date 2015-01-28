---
layout: post
title: "Intro to DNS"
tags: [dns]
---

[DNS](http://en.wikipedia.org/wiki/Domain_Name_System) is simply a naming
system  for computers so they can know how to talk to one another. The most
common analogy for DNS is a phone book. You look up the person (computer) and
the entry will tell you how to communicate with them (another computer).

I use [Hover](https://hover.com) for purchasing my domains and also as my DNS.
I would like to share how I was able to host my static site on [Github
Pages](https://pages.github.com/) but then also have a subdomain point to my
[Heroku App](https://www.heroku.com/). Let's say you have a domain foo.com and
would like to have your foo.github.io content served when you go to foo.com.
You simply follow these
[steps](https://help.github.com/articles/setting-up-a-custom-domain-with-github-pages/).

So far so good. You now have your static site deployed from your foo.github.io
repository. Now you have your "bar" repository on Github that you would like to
deploy to Heroku. Once you have deployed your app to Heroku, they have
[docs](https://devcenter.heroku.com/articles/git) for how to get started and
tutorials many languages, you will have bar.herokuapp.com.

Now in order to serve that up via bar.foo.com, you will need to do a couple of
things. You will first need to setup a CNAME record with Hover. Simply add a
CNAME record via the DNS tab in your admin console for hostname="bar", record
type="CNAME", and value="bar.herokuapp.com". You are good to go with your
domain configuration.

After all that is done, you will go back to Heroku and you can run the
following command from your terminal `heroku domains:add bar.foo.com`. This
will configure your bar.herokuapp.com to be served via bar.foo.com.
