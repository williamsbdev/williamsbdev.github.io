---
layout: post
title: "Testing Django with Codeship.io"
tags: [django]
---

In my previous [post] I set up a simple Django project. The next thing I will
do is create a [Github repo] and push up code. Now the next step before I
actually write any more code is running the tests in an automated fashion every
time I push to master. I may forget to run all the tests when I am working
locally but I can set it up to run on every push to Github. To do this, there
are several options, [travis-ci], [snap-ci], and [codeship]. I will focus only
on codeship as that is the one that I have chosen for this project.

After setting up an account, codeship has a flow to get the first project
setup. I started by connecting my Github account so that I could access my
projects. In the next step, I select the [project] that I would like to build.
Finally, I configure the project with the following for the `Setup Commands`:

```bash
pip install -r requirements/test.txt
# Sync your DB for django projects
python manage.py test syncdb --noinput
python manage.py test migrate --noinput
```

The second box `Configure Test Pipeline` should like like this:

```bash
# Running your Django tests
python manage.py test
```

At this point, the project is setup and waiting for your first push the repo to
kick off the first build.

[post]: http://williamsbdev.com/posts/setting-up-django/
[Github repo]: https://github.com/code-camp-api
[project]: https://github.com/code-camp-api
[travis-ci]: https://travis-ci.org/
[snap-ci]: https://snap-ci.com/
[codeship]: https://codeship.io
