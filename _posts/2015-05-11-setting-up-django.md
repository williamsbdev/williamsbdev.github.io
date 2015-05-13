---
layout: post
title: "Setting up Django"
tags: [django]
---

It has been a long time since I've done a Django project. Here are my notes on
how to get started. I already have my machine setup with the dev environment
ready to go by running my [installer].

```bash
mkvirtualenv <project name>
pip install django
mkdir <project name>
django-admin.py startproject project
mv project/manage.py project/project <project name>
```

At this point, I should have a fully functioning app. I am creating an example
project for myself with [code-camp-api]. After this, I went through and removed
all the comments that came for free in the scaffolding. After this, I created a
`requirements.txt` and a `requirements` directory. The directory contains a
file for each environments dependencies as they may differ. The
`requirements.txt` points to the `requirements/production.txt`.

After this I created a `settings` directory and moved the `settings.py` file to
`settings/base.py`. Then I created a `settings/test.py` and a
`settings/production.py`. In addition to this, I also had to modify the
`wsgi.py` and the `manage.py`. I have the `wsgi.py` always point to the
`production.py` settings file as I will only use the `wsgi.py` to deploy to
production. The `manage.py` on the other hand, will take the environment as the
first parameter after `./manage.py` (ie `./manage.py test test`).

At this point, I still have a functioning app that is now ready for multiple
environments.


[installer]: https://github.com/williamsbdev/osx-workstation
[code-camp-api]: https://github.com/williamsbdev/code-camp-api
