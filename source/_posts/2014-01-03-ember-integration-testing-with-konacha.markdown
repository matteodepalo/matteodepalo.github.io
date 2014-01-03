---
layout: post
title: "Ember integration testing with Konacha"
date: 2014-01-03 17:00:36 +0100
comments: true
categories: [ember, testing, integration, konacha, mocha, chai]
---

Ember is truly an awesome framework. The exciting community and the quality of the code has brought joy to front-end development.
That said, testing (more importantly integration testing) is a part of the framework that isn't quite there yet, and for various reasons:

- Lack of well defined best practices
- Few complete examples
- Debugging issues during tests is hard

## Problems

The [guide][0] on the Ember website is a good start but it's not enough. It won't tell you anything about how to handle the run loop during tests, how to work with timers or how to configure the store.
If you look around for examples they are either outdated or don't work with the framework you are using (Mocha, QUnit).

## Towards a viable stack

After spending some time trying and failing I believe I've reached a stack that makes me happy and that I consider solid enough:

- Rails (asset pipeline)
- Konacha (Chai and Mocha)
- Sinon

Rails might seem an overkill for just the asset pipeline, but currently it's the most convenient way to build Ember applications. I've tried [ember-app-kit][1] and although it's going in the right direction with the ES6 module system-aware resolver, it still has some rough edges like slow compilation times and a vast API surface.

Once you go with Rails you can draw from a nice pool of libraries built around it. [Konacha][4] is one of them.
If you too think that testing like this is cool, keep reading:

<iframe width="771" height="434" src="//www.youtube.com/embed/heK78M6Ql9Q" frameborder="0" allowfullscreen></iframe>

Konacha uses [Mocha][2] and [Chai][3] in combo. These libraries will make if you feel at home if you're coming from the RSpec world.
It also spins up a web server on the port 3500 that you can visit to run your tests (don't worry there is still a command for your CI).

The main problem is that Konacha uses Mocha to run tests and Ember supports only QUnit for integration testing out of the box; however teddyzeenny built an [adapter][5] for this purpose. Include it in the spec_helper file like this:

```coffeescript
#= require sinon
#= require application
#= require ember-mocha-adapter

Ember.Test.adapter = Ember.Test.MochaAdapter.create()
App.setupForTesting()
App.injectTestHelpers()
```

Now you can use Ember test helpers like `visit` or `click` without worrying about asynchronous behavior. Just chain them or call `then` if you want to execute some code after asynchronous actions have been performed.
For example:

```coffeescript
describe 'Notices - Integration', ->
  beforeEach ->
    visit('/')

  it 'adds a Notice to the list', ->
    fillIn('input[type="text"]', 'test')
    .click('input[type="submit"]').then ->
      find('.title').text().should.equal('test')
```

There are some other important things to add to the spec_helper file:

```coffeescript
mocha.globals(['Ember', 'DS', 'App', 'MD5'])
mocha.timeout(500)
chai.Assertion.includeStack = true
Konacha.reset = Ember.K

$.fx.off = true

afterEach ->
  App.reset()

App.setup()
App.advanceReadiness()
```

The first 4 lines will make Konacha play nicely with Ember. They will tell it to ignore leaks on globals and to avoid clearing the body of the application after each test, which is something that Ember doesn't like.

Removing animations is always a good idea during testing, it will improve speed and cause less accidental problems.

We also tell Mocha to reset the App after each test, which will destroy and reload everything bringing the router to its initial  status.

The last lines are important if you have to setup your application before loading it. When you visit localhost:3500 Konacha will load the page and Ember will run App initializers and advance App readiness on document ready. In order to have full control over this process remember to add `App.deferReadiness()` at the end of the application.coffee file, after creating the App.

If you need to perform some setup before resetting (`setup` is a custom method I've added), override the reset method like this:

```coffeescript
window.YourApplication = Ember.Application.extend
  setup: ->
    # some setup code

  reset: ->
    @setup()
    @_super()
```

## Under the hood

There are some things that this spec_helper will do under the hood. First of all it will set `Ember.testing` to `true`. This will stop the auto-run feature of the Ember RunLoop during tests to give you control over what can run with async side effects and what cannot.

For example if you want to create a fixture you need to wrap it in a `Ember.run` block or it won't execute all the async operation that will be scheduled by the application model adapter, like this:

```coffeescript
Ember.run ->
  notice = App.__container__.lookup('store:main').createRecord('notice', { title: 'test' })
  notice.save().then ->
    # check something
```

The `Ember.Test.MochaAdapter` will also enable the [bdd][6] interface for you, so you can use stuff like `describe` and `it` during tests.

I strongly suggest to read about how the Ember loop works  because sooner or later you will need that knowledge in order to debug tests. There is a good SO [answer][7] about it.

## Gotchas

Beware of timers. If your application has long running or self scheduling timers, every function that uses `wait` under the hood, like `visit`, will never resolve. It has been discussed that you should be able to explicitly avoid waiting for specific timers during tests, but in the meanwhile you can use the following hack:

Before:
```coffeescript
Ember.run.later(this, ->
  # execute something
, 1000)
```

After:
```coffeescript
setTimeout(=>
  Ember.run(=>
    # execute something
  )
, 1000)
```

This way you won't use the Ember internal setTimeout (which is not optimal), but you won't risk of executing async code outside of the run loop while allowing your tests to pass.

## Conclusion

Ember is still a relatively young framework which means that you will have to work more to get simple stuff done. However I believe the community is very conscious of this and it's pushing towards a common and strong approach for getting started quickly and testing.

[0]: http://emberjs.com/guides/testing/integration/
[1]: https://github.com/stefanpenner/ember-app-kit
[2]: http://visionmedia.github.io/mocha/
[3]: http://chaijs.com/
[4]: https://github.com/jfirebaugh/konacha
[5]: https://github.com/teddyzeenny/ember-mocha-adapter
[6]: http://visionmedia.github.io/mocha/#interfaces
[7]: http://stackoverflow.com/questions/13597869/what-is-ember-runloop-and-how-does-it-work