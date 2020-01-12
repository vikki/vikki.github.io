---
layout: post
title: Acceptance Testing with Expo
date: '2020-01-10'
description: How to run acceptance tests using appium whilst running a local build.
categories: [react-native, appium, expo, acceptance-testing, testing, android]
---

#### tl;dr - Acceptance testing an Android-based Expo local build with appium requires you to set up capabilities that start the app using a custom intent, rather than the standard activity intent. [Skip to how to do it](#doit)

![Launch an app within an app with Intent-ception]({{ site.url }}/images/inception.gif "Launch an app within an app with Intent-ception")


I'm currently working on an app in my day job that uses [Expo](https://expo.io), which is a tool that manages the native code aspects of react-native, allowing the developer to focus on just the react-based parts. This can speed up development as all the development can happen in react-style javascript, and be written once for both Android and iOS. The developer also no longer needs to know Swift/ObjectiveC/Java just to build the wrapper.

Whilst there are clear advantages to having Expo take care of some of the development for you, it throws up some challenges if you want to acceptance test the app whilst its being developed. I wanted to share how I got this working for Android, as I knew it better and I've solved the issues there, but I'm fairly confident the principles still apply to iOS builds, and I'll update this if I get that working.

### How do you start an app with appium?

Normally, when running an acceptance test with Appium, you setup a capabilities object that describes the app that you want to connect to, and then issue commands explaining how to use the app. This works well for testing against released Expo apps, as by the time they're released they are deployed as APKs, just like any other Android app. 

Normal capabilities objects look a bit like this:
{% highlight json %}
{
    "pkg": "is.vikki.awesomeApp",
    "activity": "MainActivity"
}
{% endhighlight %}

However, when Expo is running in debug mode, the app that you're developing is actually launched from the Expo client app on the phone, so telling Appium to start your APK will not work (you may not even have your own APK at this point!). It is possible to create an APK during development, but that happens through Expo's servers, rather than locally, and you would have to do a full build every time you want to test your changes, taking several minutes. That kind of delay makes continuous testing during development quite frustrating, so I was hoping there would be a better way. (I'm also trying to convince my team that testing can provide value with minimal overhead, so a strong developer experience is key). 

### How *do* you start an Expo debug app?

This flummoxed me for a while, but I eventually realised that the Expo CLI is able to programatically launch the app on the device, so it was worth checking how it does it to figure out if we can replicate it with appium. I hunted down the code in [xdl](github.com/expo/xdl) that is used to launch the Expo app on Android, and discovered that expo apps-under-development are launched using an intent with a URI that points to the Expo bundler server. This command looks a bit like this:

{% highlight bash %}
adb shell am start -a "android.intent.action.VIEW" -d "exp://127.0.0.1:19000"
{% endhighlight %}
### How do you translate the adb command back into appium capabilities?

Handily, the Android driver for appium also uses `adb` (or the [Android Debug Bridge](https://developer.android.com/studio/command-line/adb)) to interact with Android devices and emulators, so we should be able to translate this command into something that appium can use. It's possible to specify capabilities using the intent action in appium, but when I tried to do it, it was mandatory to also provide an activity. That would look a bit like this: 

{% highlight json %}
{
    "pkg": "host.exp.exponent",
    "activity": "host.exp.exponent.experience.HomeActivity",
    "intentAction": "android.intent.action.VIEW",
    "optionalIntentArguments": "-d exp://127.0.0.1:19000"
}
{% endhighlight %}

This is equivalent to an adb command that looks like this:
{% highlight bash %}
adb shell am start host.exp.exponent/host.exp.exponent.experience.HomeActivity -a \
 "android.intent.action.VIEW" -d "exp://127.0.0.1:19000"
{% endhighlight %}

If you run this, you'll see that it just launches the Expo client but appears to ignore the action that starts our app, as though you had pressed the Expo icon on the Applications screen on the phone. *So* close!

### How do I leave out the activity? <a name="doit"></a>

We need to be able to leave out the activity, so that it won't override the intent action and extra intent arguments that start the app-under-development. Currently, this isn't possible, but  its just due to the validation in appium preventing this combination of arguments, so I [PR-ed a change](https://github.com/appium/appium-adb/pull/475) that allows you to leave out the `activity` when you provide the `intentAction` and `optionalIntentArguments`. That means that once it's released, you will be able to use this in desiredCapabilities like this:

{% highlight json %}
{
    "pkg": "host.exp.exponent",
    "intentAction": "android.intent.action.VIEW",
    "optionalIntentArguments": "-d exp://127.0.0.1:19000"
}
{% endhighlight %}

and your app should start once your bundler is running, usually via `expo start`.

### Can I use it yet?

The fix has just been merged into appium-adb itself, but unfortunately the top level dependency hasn't been updated yet. `appium` depends on `appium-android-driver`, which in turn depends on `appium-adb`,  and while `appium-android-driver` and `appium-adb` are up to date, `appium` is still using an older dependency. I think it should be possible to use `npm shrinkwrap` to force `appium` to use the newer dependencies, but I've yet to test that, and have just been manually changing the dependency in `node_modules` until its merged.


Soon you should be able to sit back and start your Expo debug tests at the touch of a button, like this little guy. Do give me a shout at [@vikkiread](https://twitter.com/vikkiread) on twitter if you get any of this working, or if you have any comments and/or adorable gifs :)

[![Hopefully Mando doesnt keep switching them off for you]({{ site.url }}/images/baby_yoda_button.gif "Hopefully Mando doesnt keep switching them off for you")](https://www.youtube.com/watch?v=Y8bFUO939to)