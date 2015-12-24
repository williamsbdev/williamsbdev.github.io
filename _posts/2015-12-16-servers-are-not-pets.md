---
layout: post
title: "Servers are cattle, not pets"
tags: [aws, devops]
---

Martin Fowler has been quoted as saying ["If it hurts, do it more often"]. At
one job, we had a string of production deployments that did not go well. We had
great intentions of deploying every week or every two weeks. But we would
forget, or not want to deploy on Friday night, so we pushed it back. It was
"hard". One of our senior developers suggested, "Why don't we just deploy every
night?" After that, production deploys were much smoother.

## Nothing is sacred

It can be hard rolling out new software. It can be even more difficult rolling
out new servers. Since both of these tasks seem challenging lets do them early
and often. When it comes to rolling out new servers, it good to have the
mindset of "Servers are cattle, not pets". So we should have the ability to
spin up new servers any time and have it exactly the same as it was when it
went away or very close (minus software that may need to be deployed to it).

As we're making changes to the servers, or desiring to make changes to the
servers, let's make sure we do it early and often. This is the premise behind
Fowler's post. As the time between actiosn grows, so does the pain felt in
applying that change. So let's kill all our servers.

## Recording small deltas

There are tools out there that give you the ability to roll out changes to
servers (ie Chef and Puppet). These tools also record what was done so that you
can play it back if you need a completely new server. Now this seems great, but
at the end of the day, I still feel that we're treating our servers like pets.
We don't want to pull the plug out of the wall and completely kill the server.
Small incremental changes are good but it is still feels too caring and
nurturing. Many operations have chosen this route for managing servers and this
is better than doing it all manually. These tools have testing frameworks so
you can have more confidence as you're applying the deltas.

## Immutable servers

I started out believing that the tools above were THE way to manage servers. As
I've started doing more DevOps work, I've come to realize that I prefer the
more immutable nature of things like Docker containers.

["If it hurts, do it more often"]: http://martinfowler.com/bliki/FrequencyReducesDifficulty.html
