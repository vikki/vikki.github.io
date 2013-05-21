---
layout: post
title: Hello World Code Test
---
{% include header.ext %}

hey there

{% highlight javascript linenos %}
// a dsl returning a future
angular.scenario.dsl('historyLength', function() {
  return function(selector) {

    // this adds a fn into the queue,
    // whitout calling addFutureAction, it would have no effect during the test (REPLAY phase)
    // @param {DOMWindow} appWindow The window object of the iframe (the application)
    // @param {jQuery} $document jQuery wrapped document of the application
    // @param {function(error, value)} done Callback that should be called when done
    //                                      (will basically call the next item in the queuue)
    return this.addFutureAction('Parsing an id from message', function(appWindow, $document, done) {
      var historyLength,
          error = null;

      if (!window.history) {
         error = 'no history api available';
      }
      historyLength = window.history.length;
 
      // call the next step of the tests - continue
      // first argument is an error - spec will fail with this message
      // second argument is a value
      done(error, historyLength);
    });
  };
});
{% endhighlight %}
