---
layout: post
title: "CI/CD with Travis CI"
tags: [django, ember, ember-cli]
---

The first thing I want to do when I am working on a project of any kind is
setup continuous integration and continuous deployment (CI/CD). Any good build tool or
framework will have test running as a first class citizen. I like both [Django]
and [Ember].  Django has a test task built into the framework and great testing
tools. Ember has great testing tools and with [Ember-CLI], a great test task
built in. So when I start with either project, I do not have much setup to get
tests going.

To automate the tests every time I commit something, if I am working on a
private project, I like to use [Codeship.io]. If I am working on a public OSS
(Open Source Software) project, the defactor seems to be [Travis CI]. Both
services offer great integration with [Github] and [Bitbucket]. I am currently
using these two projects ([code-camp-ui] and [code-camp-api]) to learn new
things and CI/CD is one of those things.

To setup a Travis CI build, create an account, and then link either your Github
or Bitbucket account. You will then be able to select the projects that you
would like Travis CI to build. A build will be triggered for every push to the
master branch or any pull requests that are opened. Travis CI will also report
in the pull request the result of the build. Here are example [Django
.travis.yml] and [Ember .travis.yml] setup with continuous integration.

When I got to the continous deployment step of the process, I chose to deploy
the Ember application to [Amazon's AWS S3] and the Django application to
[Heroku]. In this process, I learned that Travis CI comes with a [CLI], which
is open sourced and available via this [Github Repo], that allows easy
configuration of the .travis.yml for continuous delivery after successful
builds. Deployments can get quite complicated within your .travis.yml but for
me, a successful build should deploy the software to a single place. In the
future, I may have master deploy to a staging environment and then a production
branch deploy to the production environment.

After installing the CLI, I followed some instructions on their [blog] for
setting up the deployments I mentioned above with S3. I also followed a
[friend's blog post] on how to setup an AWS user via an IAM role that I would
use with my deployment. Additionally, I followed the [docs] for a deployment
via Heroku. The Travis CLI also allows for the ability to encrypt the API keys
used for deploying the code. I was incredibly pleased with how quick and easy
it was to setup my CI/CD pipeline.

In the end with the deployment and everything, the [final Django .travis.yml]
looked like this, and the final [Ember .travis.yml] looked like this.

[Amazon's AWS S3]: https://aws.amazon.com/
[blog]: http://docs.travis-ci.com/user/deployment/s3/
[CLI]: http://blog.travis-ci.com/2013-01-14-new-client/
[code-camp-api]: https://github.com/williamsbdev/code-camp-api
[code-camp-ui]: https://github.com/williamsbdev/code-camp-ui
[Codeship.io]: https://codeship.io
[Django]: https://www.djangoproject.com/
[Django .travis.yml]: https://github.com/williamsbdev/code-camp-api/commit/3697b1e610a6566cfb0d4ae0da85dc97c56c8bc3
[docs]: http://docs.travis-ci.com/user/deployment/heroku/
[Ember]: http://emberjs.com
[Ember .travis.yml]: https://github.com/williamsbdev/code-camp-ui/commit/5e159fac265e38f7c3a2ed20c4ea0e33eb5158ef#diff-354f30a63fb0907d4ad57269548329e3
[Ember-CLI]: http://www.ember-cli.com
[final Django .travis.yml]: https://github.com/williamsbdev/code-camp-api/blob/c1d70d4fd5bd55a6854193cf9cbe49a8a4971682/.travis.yml
[final Ember .travis.yml]: https://github.com/williamsbdev/code-camp-ui/blob/bc0773afcc109d3b1a5d4332c491ba0c6fae8575/.travis.yml
[friend's blog post]: http://tayhobbs.com/ember/2015/04/15/deploying-an-ember-cli-app/
[Github]: https://github.com
[Github Repo]: https://github.com/travis-ci/travis.rb#readme
[Heroku]: https://www.heroku.com
[Bitbucket]: https://bitbucket.org
[Travis CI]: https://travis-ci.org
