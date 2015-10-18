---
layout: post
title: "Overriding Ember Addons"
tags: [ember, ember-cli]
---

Ember Addons are a great part of the Ember ecosystem. They allow for some great
extensibility. However, there are times when you want to change the
functionality/default configuration the addon provides. Let's take the
following example:

```js
// some-addon/app/initializers/some-initializer.js

import addonInitializer from 'some-addon/initializers/addon-initializer';

export default {
    name: 'some-addon-initializer',
    initializer: addonInitializer
};
```

If you notice, the `some-addon` placed this initializer in the
`app/initializers` folder of the addon. This means that the file will get
merged into the app where it's included. This is very handy as you will have
the initializer by default. This is fine until you need to change the
functionality of the initializer just ever so slightly.

In order to override the initializer, all we have to do in our host app is
create a file in `app/initializers` named `some-initializer.js` like below:

```js
// your-app/app/initializers/some-initializer.js

var addonInitializer = function() {
  // your custom code here to override addon-initializer
};

export default {
    name: 'some-addon-initializer',
    initializer: addonInitializer
};
```

This goes for any file that is in the `app/` of an addon. If you create a file
with the same name in your app, you will override the addon and have complete
control. [Here is a great stackoverflow Q/A] about doing this.

[Here is a great stackoverflow Q/A]: http://stackoverflow.com/questions/29634920/how-to-override-default-functionality-in-ember-addons
