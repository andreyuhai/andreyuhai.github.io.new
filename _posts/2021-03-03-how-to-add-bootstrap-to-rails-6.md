---
title: How to add Bootstrap to a Rails 6 project
tags: [ruby, ruby-on-rails, rails-6, bootstrap]
categories: [TIL, Ruby]
---

Create a new Rails 6 project.
```bash
$ rails new awesome-project
```

Add `bootstrap`, `popper.js` and `jquery` using `yarn`.
```bash
$ yarn add bootstrap popper.js jquery
```

Copy the snippet below into `config/webpack/environment.js`
```javascript
const webpack = require('webpack')
environment.plugins.append('Provide', new webpack.ProvidePlugin({
  $: 'jquery',
  jQuery: 'jquery',
  Rails: '@rails/ujs'
}))
```

Rename the `application.css` in `app/assets/stylesheets` into `application.scss` and copy & paste the snippet below.

```
@import 'bootstrap/scss/bootstrap';
```
