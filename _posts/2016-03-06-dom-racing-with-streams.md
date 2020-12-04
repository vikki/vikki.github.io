---
layout: post
title: Dom Racing with Streams
date: '2016-03-06'
description: Adventures in modelling async logic that interacts with each other as promises, and as streams.
categories: [learning,about]
tags: [streams, promises, rxjs, javascript]
---

*Adventures in modelling async logic that interacts with each other as promises, and as streams*

![Mario Kart - improved with kittens]({{ site.url }}/images/mariokit.gif "Mario Kart - improved with kittens")

###### Like this ^^ but with DOM elements, and sadly fewer kittens

I work on some code that polls the DOM for the presence of one or more elements. We search for each element, but we’ll only use the first one that we find, so as soon as we find an element, we stop looking for the others. The polling code itself is really easy to wrap in a promise, but what’s less simple is choosing the first one and then tidying up the others. I’d never seen a tutorial about doing something this fiddly with promises, and a colleague suggested that streams could be used to solve all our problems, so I wrote up a comparison of each style in the hope of helping somebody else and learning more about the problems that streams solve.

## Attempt 1: Promise-API

The code was originally written using callbacks, and later promises, to manage polling for elements in the DOM. You specify a selector with which to search the DOM and it does this at regular intervals, resolving when the promise is found, or rejecting if it times out.

For a single element this works really well, but our project is structured such that even when we look for multiple selectors, only one element/selector should ever win. If there are multiple selectors provided, we should look for all of them, use the first one we find, and disregard the rest.

It’s *relatively* simple to just take the first promise that resolves, and leave the rest be, but as good, efficient, web citizens it makes sense to tidy up, and stop polling for the elements we aren’t going to bother using. The real code is third party javascript so it’s only polite to get out of the way of the main webpage wherever possible, and even if it weren’t there’s really no point in continuing to poll for stuff we aren’t going to use.

To do this with an array of promises, we map over the selectors we have, creating a promise-like object for each one, representing our polling logic. When any of these promises resolve, we’ll need to stop the other ones polling. To do this we can just iterate over our polling promises and call some kind of cancel() function to get them to stop polling.

<script src="https://gist.github.com/vikki/c82ef0e584d9b92ee9c4dd80da1bf5f2.js"></script>

## Attempt 2: Patching the Perils of Promises (sorry :P)

Promises aren’t designed to be cancelled, so it’s not possible to guarantee that the fulfillment handler of one promise happens before another is resolved or rejected. As such they are inherently inoperable in this way, because 2 promises can resolve at the same time, and once a promise has been resolved it can’t be cancelled or rejected. You can test this out in the gist above by making the code at the bottom add both elements at the same time. This scuppers our plans, because we want to be able to reject all but the first promise.

We can work around it by wrapping the promise with another one. To extend our example, rather than directly relying on the result of the promise we want to reject, we can maintain the state of our choice outside the promises themselves and instead throw during the fulfillment handler.

<script src="https://gist.github.com/vikki/de5bf4a11c3ab5fc08189caf01c2e5d7.js"></script>

This in some ways is nicer; it’s more flexible (it’s a lot easier to change the number of DOM elements we want to wait for with this model), and it also more accurately represents our problem - we’re using 1 promise to represent whether or not the DOM elements exists, and another to represent whether or not we’ve chosen it. What’s less awesome is that we need to maintain some state to represent the DOM elements that we’ve chosen, or in other words, which one won.

![Wun Wun!!!!!one1!]({{ site.url }}/images/wunwun1.jpg "Wun Wun!!!!!one1!")

##### *Which Wun Wun? !!!!!one1!*

## Attempt 3: Streams Will Save Us

Refactoring the code into streams simplifies the logic quite a bit. The polling logic works in much the same way; we create a stream for each DOM element we’re interested in, and instead of resolving the promise we advance the stream.

I’m using RX.js in the following example as I did it primarily as a learning exercise, and RX has lots of helper functions to help me muddle my way through, but the principle should work for any FRP library, and were I to move this into our production code I’d probably use something a little smaller.

<script src="https://gist.github.com/vikki/0eb5bbef5cd1334e2b4a1e5039ad33a5.js"></script>

Where it gets pretty is that we can merge the streams, and then it’s possibly to replace our fiddly cancellation logic with a simple, stateless filter. What’s particularly cool about this is that we don’t need to explicitly cancel the polling at all any more 😀 Our stream is clever enough to know that once we’ve taken what we needed from the stream, it can dispose all of the connected streams. This means that all we need to do is tell our polling stream to stop polling when it’s disposed, and then once we’re done with the consuming streams, it will stop polling automatically. This is not only awesome, it also makes our code much easier to read. It also feels cleaner that the polling code decides when it should tidy up, and it isn’t the responsibility of the calling code to tell it to stop.

![Spengler & Venkman are not fans of RX.js]({{ site.url }}/images/dontcrossthestreams.jpg "Spengler & Venkman are not fans of RX.js")

##### *Actually, **do** cross the streams :P*

In summary, streams made this particular part of the code much prettier and seem to be a cleaner way of representing our model. I was particularly impressed by how intuitive refactoring into streams was; before attempting this I’d written code that was intuitively “stream-y”, like reading a file, but I’d never tried to refactor something that was a little less obviously stream-like. Once I got my head around the idea that the entire chain needs to be stream-based, so that the disposal logic is cleaned up, the code all but wrote itself.

Actually using the stream implementation is probably overkill for what we would use it for right now, as RX.js is a little large to be worth the refactor. However, I’m aware of some similar libraries with smaller footprints, and I understand that the next version of RX will be more modular <linkeh>, so if we can find a way of including it without bloating our code, it’s definitely something to consider.

## Further Reading

- [RX Marbles](http://rxmarbles.com/) - visualise React concepts
- [The introduction to Reactive Programming you've been missing](https://gist.github.com/staltz/868e7e9bc2a7b8c1f754) - excellent overview of basics
- [General Theory of Reactivity](https://github.com/kriskowal/gtor) - slightly hard to read but comprehensive overview of how promises relate to streams and much *much* more
