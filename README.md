# williamsbdev.com blog

[![Build Status](https://travis-ci.org/williamsbdev/williamsbdev.github.io.png)](https://travis-ci.org/williamsbdev/williamsbdev.github.io)

In order to run the blog locally you will need to have either Ruby or Docker on
your machine.

If changes are made to the styling or JavaScript, run the following:

    npm install
    grunt deploy

For Docker, run the following:

    docker run --rm --volume=$(pwd):/srv/jekyll -p 4000:4000 jekyll/builder jekyll serve

For Ruby, install jekyll and then run jekyll serve

    gem install jekyll

Once this has finished installing all the dependencies, run the following
command to spin up the site locally at localhost:4000:

    jekyll serve

Please visit [jekyll] for more information about how it works.

Credit for all the design work goes to [Jarrod C Taylor]. I took the design he was
using for his [blog].

[jekyll]: http://jekyllrb.com/
[Jarrod C Taylor]: https://github.com/JarrodCTaylor
[blog]: http://jarrodctaylor.com
