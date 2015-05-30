---
layout: post
title: "Deploy Django project to Heroku via Codeship.io"
tags: [django]
---

Now that we have successfully run all the tests for our Django project on
[Codeship.io], we want to deploy our application. We will first setup our
application with [Heroku] and then we will setup our Codeship.io job to deploy
to Heroku on a successful build.

To setup Heroku, create an account and an app. We will create (code-camp-api).
You can then follow the instructions to download and install the [Heroku
toolbelt]. Since we have already created this project, we will skip the step
about creating a new Git repository and jump to deploying our application.  We
will add Heroku as a remote to our git repository with the following:

```bash
heroku git:remote -a code-camp-api
```

Next we will do a deploy to Heroku to test it out. We will push our master
branch to Heroku.

```bash
git push heroku master
```

To deploy our project with Codeship, we will go back to the job we have created
([previous post]) and add continuous deployment. We want to deploy only our
master branch to Heroku. To deploy our project to Heroku, we will need to make
a few modifications to our codebase, first by adding some new files and second
by modifying the `manage.py`. The first change we will make is to add a few new
files. The first file we will add is the `Procfile`. This is a file to tell
Heroku how to run our application.

```text
web: gunicorn project.wsgi --log-file -
```

The second file we will add is a `runtime.txt`. This will tell Heroku which
python version we would like for our environment.

```text
python-2.7.9
```

The third file we will add is the `settings/production.py`.

```python
# settings/production.py

from project.settings.base import *
import dj_database_url
```

The fourth file we will add is the `requirements/production.txt`.

```text
-r base.txt
dj-database-url==0.3.0
dj-static==0.0.6
django-toolbelt==0.0.1
gunicorn==19.0.0
psycopg2==2.5.4
static3==0.5.1
wsgiref==0.1.2
```

Finally we will point our top level `requirements.txt` to our
`requirements/production.txt`.

```text
-r requirements/production.txt
```

The second change we will make is only for this application so that we can run
with different settings files. The applications that I have worked on have had
different settings files so that we can changed based on our environment.  We
will modify the `manage.py` file to allow Heroku to be able to handle our
multiple settings files. By default, we will assume that you are running with
production settings. However, if the second argument (`manage.py` is the first
argument) is `test`, we should use our test settings. This is used mainly for
our Codeship job when running the tests.

```python
# manage.py

#!/usr/bin/env python
import os
import sys

if __name__ == "__main__":
    settings_file = 'production'
    args = sys.argv
    if sys.argv[1] == 'test':
        settings_file = 'test'
        args = [sys.argv[0]] + sys.argv[2:]
    os.environ.setdefault("DJANGO_SETTINGS_MODULE", "project.settings.{}".format(settings_file))
    from django.core.management import execute_from_command_line
    execute_from_command_line(args)
```

With all this done, you will now be able to push code to our master branch, run
the tests, and deploy to Heroku if all tests pass.

[Codeship.io]: https://codeship.io
[Heroku]: https://heroku.com
[Heroku toolbelt]: https://toolbelt.heroku.com
[previous post]: http://williamsbdev.com/posts/testing-django-with-codeship-io/
