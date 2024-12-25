---
title: "Appium VS Calabash"
author: elgris
date: "2016-03-12"
description: "My experience of picking appropriate test tool for mobile application"
categories:
    - "tech"
---

OK, what do we have here? A mobile application written with Cordova for 2 platforms: iOS and Android. I'm lazy, I don't want to check that all critical functionality works as expected after each release. In 2 platforms. And I don't want anyone to do that, because manual work is stupid in IT. So I need a proper tool to test both applications automatically, so I have enought free time to explore the world around me :)

# TL;DR

Decided to go with [Calabash](http://calaba.sh) at the end.

# Constraints

All right, let's define following constraints:

 - I want to write tests once (or at least write as less code as possible :) ) and run them against both Android and iOS builds.
 - I need to test **hybrid** application created with [Cordova](https://cordova.apache.org/).
 - I'd like to write tests in Javascript, because in [SeeSaw Labs](http://www.seesawlabs.com/) we use plenty of it, so many people can support the tests.
 - Tests must be readable. Ideally, written in Gherkin, but some framework that provides [frisby](http://frisbyjs.com/)-like syntax is OK also.

There is not so many frameworks that satisfy constraint 1. So I ended up with two of them: [Appium](http://appium.io) and [Calabash](http://calaba.sh).

# What's good about Appium?

- No need to link any additional code to your application. Just build it, run it (emulator is fine), Appium will do the rest.
- Easy debugging. Appium uses WebDriver API which can be used with any HTTP client (`curl` to the rescue).
- Fancy UI that allows you to investigate layout of pages, prepare queries to the elements and even record test scenarios like Selenium IDE.
- Integrates with CucumberJS very easy. Basically, you just write your tests with CucumberJS and communicate to Appium with webdriver client.
- If you know how to write tests for Selenium WebDriver - you already know how to use Appium.

# What's wrong with Appium?

- Doesn't work well with hybrid applications on Android. In my case it was impossible to determine an element on page. The coordinates of an elements were changing each time I scrolled the page.
![ where is the EditText element you say? o_O !](/img/2_appium_vs_calabash/android.png)

- Doesn't work well with Cordova app on iOS with iOS SDK that comes with Xcode 7.2. The problem is: somehow the elements (buttons, labels, inputs) are "disabled", so Appium just does not see them and cannot interact with them. Look at the screenshot. Appium Inspector was told not to show "disabled" elements. Dude, where's my button then? :)
```
info: [debug] Responding to client with error: {"status":7,"value":{"message":"An element could not be located on the page using the given search parameters.","origValue":""},"sessionId":"eccc5b20-a5dd-4f41-9ce8-c16964e3f64c"}
info: <-- POST /wd/hub/session/eccc5b20-a5dd-4f41-9ce8-c16964e3f64c/element 500 754.829 ms - 179
```
There is an issue about that: https://github.com/appium/appium/issues/4131

- Documentation is scarce, I spent lots of time to deliver first test.

# What's good about Calabash?

- It works. It just works with hybrid app on both platforms. Of course, there is some diference between APIs for Android and iOS, but everything is well documented. There is even examples that cover more 80% of your needs: https://github.com/calabash/x-platform-example
- There is a good tutorial about linking Calabash library to iOS build: https://github.com/calabash/calabash-ios/wiki/Tutorial%3A--Creating-a-cal-Target
- Cucumber out of the box.
- Command-line tool that allows debugging: `calabash-android console` and `calabash-ios console`. That's how I inspected application pages and debugged interaction with elements using selectors.

# What's wrong with Calabash then?

- Ruby only :(
- Need to prepare special target to build iOS application
- No fancy UI which could help a lot with finding proper selector of the element

# What's at the end?

I decided to go with [Calabash](http://calaba.sh). Despite it does not have Javascript bindings (yet), writing Ruby code was easy and fun. Actually, by the end of first day I have successfully set up all required infrastructure, ran first test on both platforms and prepared a tutorial for the rest of the team.

I would recommend Appium to those who makes native applications. With hybrid apps (at least Cordova-based) I ran into bunch of problems. Perhaps those problems could be solved, but **learning Ruby was much faster** than solving those artificial problems.