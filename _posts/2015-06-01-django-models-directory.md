---
layout: post
title: "Creating Django models in a directory"
tags: [django]
---

When I am organizing my code, I like to separate objects into files and group
them based on responsibility. With Django out of the box, they expect all the
models to be in the `models.py` file. This is less than desirable as large apps
can have many models and this file can get out of hand. I like to separate my
models into their own files but there is a little work involved.

In Django versions < 1.7, there were two requirements to get models to be
recognized. First, in the model itself, an `app_label` was required in the
`Meta` class. The app_label was required to be the name of the installed app
(directory above the models directory). For example:

```python
# code_camp/models/speaker.py

from django.db import models

class Speaker(models.Model):
    name = models.CharField(max_length=100)

    # only required in Django versions < 1.7
    class Meta(object):
        app_label = 'code_camp'
```

The second requirement is that the model needs to be in the models module (the
`__init__.py` in the models directory). In Django versions >= 1.7, this is the
only requirement to register the model with the installed app.

```python
# code_camp/models/__init__.py

from .speaker import Speaker as _Speaker
```

I like to import the `Speaker as _Speaker` because this prevents people from
simply importing the model like so:

```python
# will not work
from code_camp.models import Speaker

# only way to access Speaker model
from code_camp.models.speaker import Speaker
```

Since Python is an explicit language, I prefer to import the object from the
original location instead of through another import.
