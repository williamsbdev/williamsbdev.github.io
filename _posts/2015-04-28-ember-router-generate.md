---
layout: post
title: "Ember.Router.generate"
tags: [ember]
---

At work we had a story where we wanted an error state to have a link to do a
hard page refresh to get rid of potentially bad state on the page. We were
optimistically rendering the result before the promise had resolved. If the
promise came back in a failed state, we would display a global error message as
the user was transitioned away and may have even navigated further away. The
desired outcome would be that the user would just refresh the page and stay
exactly where they were. This would clear our in memory objects and fetch the
clean data from the server, resetting the app. In order to keep the user on the
page they were currently on, we did was the following:

```javascript
// app/controllers/application.js
import Ember from "ember";

export default Ember.Controller.extend({
  actions: {
    reload: function() {
      var router = this.container.lookup("router:main");
      window.location.hash = router.generate(this.get("currentPath"));
      window.location.reload();
    }
  }
});
```

```text
{% raw %}
// app/templates/application.hbs
<div id="global-error">
  Some bad happened. Please refresh the page.
  <button {{action "reload"}} id="reload-button">Refresh</button>
</div>
{% endraw %}
```
