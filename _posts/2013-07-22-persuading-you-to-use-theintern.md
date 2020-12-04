---
layout: post
title: Persuading you to use theintern
date: '2013-07-22'
description: What is theintern? What's cool about it? What's it for?
categories: [learning,about]
tags: [testing, acceptance-testing, javascript, node, theintern]
---

Intern allows you to write both unit tests and acceptance tests, and its a nice test runner for x-browser unit tests, but its been possible to do that for a while with awesome tools like [bunyip](https://github.com/ryanseddon/bunyip). What I'm most excited about is their API for writing acceptance tests, which allows you to create BDD-style, promise-based tests in javascript, and then configure them separately to run against as many browsers as SauceLabs can provide. This post will be an overview of what it is and what it does, which I am hoping to follow up with some details on how to get started. Until I get round to that though, check out their [wiki](https://github.com/theintern/intern/wiki) and [tutorial](https://github.com/bitpshr/intern-tutorial) .

## Can't you just use selenium?
Up until theintern came along, I was writing Java-based acceptance tests in Selenium, running against local browsers and SauceLabs for more complete coverage, particularly IE and mobile. These are pretty readable (we have a dev who's new to Java, and he's perfectly happy reading them, which I think is a good sign!), and with a fair bit of wrangling we were able to hook these up to run against SauceLabs on demand. 

There were a few problems with this approach though. The biggest hurdle so far has been the classic non-deterministic nature of the tests. With a few projects now we've found that reaching a certain number of tests has caused the reliability of the tests to drop dramatically, and whilst there have been many reasons for this (DNS, async. issues, badly structured tests) its pretty annoying to have to stop committing/deploying to track this stuff down, and its also pretty challenging, not too mention demoralising, to track down these errors as they're non-deterministic.

Using Java-based selenium made a lot of sense for us in the past, given a lot of our code was Java-based, and we were coming at web development from a Java point of view. It was also reasonable given we had a large project that included Java code, with a deployment tool chain that was also Java-centric (lots of maven and ant). When we split out the front-end part of the project, and started using the life-changing grunt, our release workflow was streamlined but we still ended up having to shell out from grunt to run maven in java, which isn't the prettiest or most maintainable solution.

Finally, whilst the Selenium Java tests themselves are pretty approachable to a non-Java developer, I fear that maintaining the infrastructure required to connect the tests up to Sauce Labs or local browsers would actually end up being quite challenging. Furthermore, even as an enthusiastic polyglot, it just seems like extra overhead / avoidable context-switching to write tests in one language and implementation in another. And don't get me started on trying to get numbers out of Javascript in Java.

So, apart from the non-deterministic stuff, we were fairly happy with our Selenium tests, but our workflow was far from perfect. Enter theintern.io :)

## What is it?
theintern is a runner and functional testing framework, allowing you to write unit and functional tests as AMD-style modules in node, and then run them against real browsers, and/or, in the case of the unit tests, headless webkit. Its possible to use pretty much whatever you want for assertions, so any pre-existing chai madskillz may be leveraged. The real awesome-osity comes though, I believe, from the ability to write promise-based functional tests using the web driver API to interact with the browser, which can then be chucked at All The Browsers via Sauce Labs.

## So, why's it good?
Number one fave. thing : because you're writing stuff that heavily uses the promise-based API you end up with really readable tests. I would go so far as to say that they're pretty. I appreciate that sounds trivial, but you really want your tests to be as clear as possible; ideally they should be idiot proof, as I have often found that Future Me can be an idiot and doesn't understand Past Me. If you favour the tests-as-doc strategy, this is even more important, and its a nice way of elaborating on the bdd-style test names, if that's your thing. Along the same lines, I've always favoured erring on the side of super-simple tests as there is NOTHING more terrifying than bugs in your tests, and the chainable web driver commands give the tests great clarity. Finally, before using a library that's async from the ground up (more or less everything you do uses the promises stuff), I hadn't realised how inherently async acceptance tests tend to be. Given that the flow tends to go:

* request page
* wait for page to be ready
* interact with a thing
* wait for that to take effect
* see if it did the right thing

using a library where you write in promises makes you approach the problem in the right way, or at least a better, more reliable way, and is a tonne more readable than liberally sprinkling your selenium code with Thread.sleep()s . 

I also love that the configuration is cleanly separated from the tests themselves, which makes it easy, once you have it setup, to switch between a quick local test, trying it out on say modern desktop browsers, and a full sweep. Local testing of functional tests isn't currently very well documented, but setting up a Selenium server isn't anywhere near as scary as it sounds, and once you hook it into grunt its no trouble at all. That said the main thing that makes this so useful is SauceLabs integration; many of the other tools I've used have required user interaction to say when the browsers were ready to use, this all Just Works with theintern. Its also great to feed the runner a config file with all the SauceLabs details, as specifying the browser set can get pretty unwieldy from the command line. 

Other excellent-ness includes grunt, npm and travis support out of the box. 

## But nobody's perfect…
Currently if your code errors, the errors aren't very helpful (no line numbers and only very generic errors), so it can be tough to track down what and where the issue is. This is because you're reading in some js, and either executing that in node, or even more confusingly, executing it in a (remote) browser, but would really benefit from more descriptive errors with complete stack traces. The dynamic nature of the tests (reading in files, and timeouts) can also hinder debugging with something like node-inspector. The usual caveats of promises being confusing as sin to beginners also apply; they're very readable but can be a bit weird to debug, at least at first. Finally web driver error messages aren't always super-helpful; most of the time its things like missing elements and timing issues, but if you get the dreaded 13 UnknownError you're kind of on your own ([this](https://code.google.com/p/selenium/wiki/JsonWireProtocol) will help) . Also as quite a new library, there aren't that many tutorials just yet, but I'm hoping this will change. 

Another potential issue is that modules are defined in AMD, whilst node modules are more commonly written in commonjs style.  Its still quite possible to use node modules in your intern tests (at least I haven't found any issues yet), but the syntax for including them is a little strange, and obviously you're having to learn another way to define modules if you're only used to commonjs.

## Final Verdict
I think its pretty badass, but I would say that, I wrote a whole blog post on it :P It can be challenging to get started, given the requirements to understand AMD, and promises etc., but I think the investment is worth it to achieve more maintainable, reliable , cross-platform test coverage.

