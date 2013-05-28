---
layout: post
title: Extending the karma api
---
{% include header.ext %}

# Extending the karma api

#### tl;dr - karma uses futures to drive your tests, so if you want to extend their API, you'll need to as well. Code sample [here](#howdoesthathelp)

So, we've started using angular.js for a project in work, and whilst we've mostly found it very shiny, we've run into some hiccups getting to grips with the testing side of things. Its pretty straightforward to write basic tests by modifying the examples in the docs, but eventually, you may need to do something more complicated, actually understand how it works, or just get cocky and try to do more advanced things :P For the uninitiated (read:me), the way that you go about doing non-built-in things wasn't obvious, and I couldn't find any docs on how to go about doing it, save for a [life-saving gist](https://gist.github.com/vojtajina/1840093) by Vojta Jina himself, so here goes my attempt at explaining how you do it. n.b. I have no idea if this is the correct/idiomatic way to do it, I just know it works, but would massively appreciate any comments!

## What's the problem? <a name="wat"></a>
Say you want to have a test perform a comparison on something that *is* available in the browser (I'm just going to flat out ignore x-browser compatibility to keep this short, but you may want to think about it :P), but isn't available via the karma api. This happened to us as we wanted to verify that angular didn't affect the browser history, and figured that the history api might be a cool way to do this. 

Say we define this in pseudo code like so :
- do some stuff which could add to the browser history
- verify the current browser history is 1

We might try to implement it like this : 
{% highlight javascript linenos %}
it('does some stuff without affecting the browser history', function() {
      // do some stuff which could add to the browser history
      element('#shiny').click();

      // verify the current browser history is 1
      expect(window.history.length).toBe(1);
});
{% endhighlight %}

That's all well and good, except for the bit where it doesn't actually work. :( After some serious poking around in the karma code, I discovered that expect takes a future on the left hand side (you might say you have to pass dee future on dee left hand side, but I digress :P). In fact, to clarify, it will run, and won't chuck any errors, but won't behave how you might want it to, again because of the way karma runs your tests. Karma will set up your test using futures, which includes both the expect statements, and any interactions driven from your test (for example telling the test to register a click on the element, like we're doing here). All that happens on the first pass of the test is the creation of these futures, defining the test run. At the end of the test, karma will resolve those futures, in the order in which they were defined, and *that* is when the interactions and verifications actually happen.

So our previous test will actually do the following:
- setup the do-ing of some stuff which could add to the browser history
- inspect the browser history
- actually do some stuff which could add to the browser history
- verify the old browser history length (from before we do the stuff that might add to the history!)

What's more, as karma is expecting a future on the left hand side, its going to do future-y stuff to whatever you pass in, which in our case is actually an int, so that isn't gonna fly, and will ultimately result in the LHS of our comparison being undefined. ARGH!

So, basically we're doing stuff out of order, and using the API incorrectly (its quite interesting that its so readable/intuitive that you can actually write several tests before you start thinking of it as an API, what are the params etc.). Simple fix is to err, stop doing stuff out of order, and copy the way that the framework itself does this, as they probably have a decent idea of how this is supposed to work.

## How do you actually do that? <a name="how"></a>

The approach we took to fix this was one that's taken from how angular itself defines its interaction DSL for acceptance testing. A common (OK, its not super common but it makes a good example) use case for testing is to ensure we have the right number of certain elements. We can define this in a test like this :

{% highlight javascript linenos %}
    it('has only one neo element', function() {
        expect(element('#neo').count()).toEqual(1);
    });
{% endhighlight %}

So here we're saying, grab elements matching our '#neo' selector, count them, and expect that count to be exactly 1. We can see how this works by checking out angular-scenario.js, where the good stuff is defined. Below is a stripped down version of what's in ~1.0.6...

{% highlight javascript linenos %}
angular.scenario.dsl('element', function() {
  var chain = {};

  chain.count = function() {
    return this.addFutureAction("element '" + this.label + "' count", function($window, $document, done) {
      try {
        done(null, $document.elements().length);
      } catch (e) {
        done(null, 0);
      }
    });
  };

  return function(selector, label) {
    this.dsl.using(selector, label);
    return chain;
  };
});
{% endhighlight %}

So, here angular is creating an (angular) element constructor as part of its scenario DSL, and returning an object that defines the helper function `count()`. The pertinent bit here is that in count we don't directly do anything (probably not a surprise at this point!), rather we set up a future, passing in the future's behaviour as the second argument. The future can do whatever we want (within reason, as applied to js #butiwantapony), but ultimately it needs to call `done()` using the node-y callback convention of `done(error, result)`. In `count()` we just pass in a null error cos nothing's gone wrong, and the number of elements we've found that match the selector. 

Going back to our test again, we can see that the output of `count()` is passed into expect, which as we've discussed, resolves the future (runs that function) and chucks the output (which is now something less complicated like an int) to the `toEqual()` matcher, ready to be verified.

{% highlight javascript linenos %}
    it('has only one neo element', function() {
        expect(element('#neo').count()).toEqual(1);
    });
{% endhighlight %}

## How does that help me? <a name="howdoesthathelp"></a>

So now we know how karma sets up interactions we can use this API to write our own and verify whatever we want. `addFutureAction` is defined on the karma SpecRunner, so we can actually set up the futures wherever we want, but it can be cleaner to encapsulate them in the DSL. Going back to our original problem, we can look at the history api length via futures like this :

{% highlight javascript linenos %}
angular.scenario.dsl('historyLength', function() {
  return function(selector) {

    return this.addFutureAction('history length', function(appWindow, $document, done) {
      var historyLength,
          error = null;

      if (!window.history) {
         error = 'no history api available';
      }
      historyLength = window.history.length;
 
      done(error, historyLength);
    });
  };
});
{% endhighlight %}

We can then define our test as before, with a minor tweak on line 6 to use the DSL :

{% highlight javascript linenos %}
it('does some stuff without affecting the browser history', function() {
      // do some stuff which could add to the browser history
      element('#shiny').click();

      // verify the current browser history is 1
      expect(historyLength()).toBe(1);
});
{% endhighlight %}

and now it should actually work, both in terms of ordering and not guessing how the API works ;)

# Yay <a name="yay"></a>

So hopefully that goes some way to explaining how the karma API works, and how to work with it to solve non-standard problems. If you have any questions/comments/heckling please [say hello @vikkiread](https://twitter.com/vikkiread)
