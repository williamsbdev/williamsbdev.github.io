---
layout: post
title: "Ember CLI addons with test-support"
tags: [ember-cli]
---

I recently learned about a feature of [ember-cli] that allows for an addon to
include content only in the test environment. [Brian Cardarella] pointed out in
one of the addons that I work on that the test helper functions were getting
included into the main build of any app including this addon. Since these were
truly test helper functions, we required the functions exclusively in testing.

If you desire to only have content from your addon accessible in testing, you
can put all the content into the `test-support/` directory. When the addon is
included into the main app, it will get put into the `tests/` directory of the
including app and not included when doing a build other than test.

[Brian Cardarella]: https://github.com/bcardarella
[ember-cli]: http://ember-cli.com
