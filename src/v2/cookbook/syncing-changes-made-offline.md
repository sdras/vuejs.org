---
title: Syncing Changes Made Offline
type: cookbook
order: 3
---

## The Importance of Syncing Changes Made When the User is Offline

It's a good idea to think about scenarios when your user is either offline or has poor connectivity. This article won't cover all uses for Service Workers, but because Vue deals with changes made by users dynamically so well, in this recipe, we'll explore 

## Simple Example




## Real-World Example: Replacing Vue Resource with Axios

Let's say you're replacing the [now-retired Vue Resource](https://medium.com/the-vue-point/retiring-vue-resource-871a82880af4). You really enjoyed accessing request methods through `this.$http` and you want to do the same thing with Axios instead.

All you have to do is include axios in your project:

``` html
<script src="https://cdnjs.cloudflare.com/ajax/libs/axios/0.15.2/axios.js"></script>

<div id="app">
  <ul>
    <li v-for="user in users">{{ user.name }}</li>
  </ul>
</div>
```

Alias `axios` to `Vue.prototype.$http`:

``` js
Vue.prototype.$http = axios
```

Then you'll be able to use methods like `this.$http.get` in any Vue instance:

``` js
new Vue({
  el: '#app',
  data: {
    users: []
  },
  created () {
    var vm = this
    this.$http.get('https://jsonplaceholder.typicode.com/users')
      .then(function (response) {
        vm.users = response.data
      })
  }
})
```

## The Context of Prototype Methods

In case you're not aware, methods added to a prototype in JavaScript gain the context of the instance. That means they can use `this` to access data, computed properties, methods, or anything else defined on the instance.

Let's take advantage of this in a `$reverseText` method:

``` js
Vue.prototype.$reverseText = function (propertyName) {
  this[propertyName] = this[propertyName].split('').reverse().join('')
}

new Vue({
  data: {
    message: 'Hello'
  },
  created: function () {
    console.log(this.message)    // => "Hello"
    this.$reverseText('message')
    console.log(this.message)    // => "olleH"
  }
})
```

Note that the context binding will __not__ work if you use an ES6/2015 arrow function, as they implicitly bind to their parent scope. That means the arrow function version:

``` js
Vue.prototype.$reverseText = propertyName => {
  this[propertyName] = this[propertyName].split('').reverse().join('')
}
```

Would throw an error:

``` log
Uncaught TypeError: Cannot read property 'split' of undefined
```

## When To Avoid This Pattern

As long as you're vigilant in scoping prototype properties, using this pattern is quite safe - as in, unlikely to produce bugs.

However, it can sometimes cause confusion with other developers. They might see `this.$http`, for example, and think, "Oh, I didn't know about this Vue feature!" Then they move to a different project and are confused when `this.$http` is undefined. Or, maybe they want to Google how to do something, but can't find results because they don't realize they're actually using Axios under an alias.

__The convenience comes at the cost of explicitness.__ When looking at a component, it's impossible to tell where `$http` came from. Vue itself? A plugin? A coworker?

So what are the alternatives?

## Alternative Patterns

### When Not Using a Module System

In applications with __no__ module system (e.g. via Webpack or Browserify), there's a pattern that's often used with _any_ JavaScript-enhanced frontend: a global `App` object.

If what you want to add has nothing to do with Vue specifically, this may be a good alternative to reach for. Here's an example:

``` js
var App = Object.freeze({
  name: 'My App',
  description: '2.1.4',
  helpers: {
    // This is a purely functional version of
    // the $reverseText method we saw earlier
    reverseText: function (text) {
      return text.split('').reverse().join('')
    }
  }
})
```

<p class="tip">If you raised an eyebrow at `Object.freeze`, what it does is prevent the object from being changed in the future. This essentially makes all its properties constants, protecting you from future state bugs.</p>

Now the source of these shared properties is more obvious: there's an `App` object defined somewhere in the app. To find it, developers can run a project-wide search.

Another advantage is that `App` can now be used _anywhere_ in your code, whether it's Vue-related or not. That includes attaching values directly to instance options, rather than having to enter a function to access properties on `this`:

``` js
new Vue({
  data: {
    appVersion: App.version
  },
  methods: {
    reverseText: App.helpers.reverseText
  }
})
```

### When Using a Module System

When you have access to a module system, you can easily organize shared code into modules, then `require`/`import` those modules wherever they're needed. This is the epitome of explicitness, because in each file you gain a list of dependencies. You know _exactly_ where each one came from.

While certainly more verbose, this approach is definitely the most maintainable, especially when working with other developers and/or building a large app.
