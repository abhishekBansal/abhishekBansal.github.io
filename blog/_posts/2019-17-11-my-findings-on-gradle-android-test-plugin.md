---
layout: post
title:  "My Findings on Gradle Android Test Plugin"
date:   2019-11-17 8:00:00
author: Abhishek
categories: Android, Testing, UI Testing, Espresso
---

*Originally published at [My Findings on Gradle Android Test Plugin](https://medium.com/@discover.ab/my-findings-on-gradle-android-test-plugin-87ba973e6965)*


In my [last article](https://medium.com/swlh/espresso-unit-testing-and-things-to-watch-out-in-android-b5435a91c677) I listed out some of the issues that we faced when trying to setup `AndroidX Testing Framework` with [Espresso](https://developer.android.com/training/testing/ui-testing/espresso-testing). While there were many issues, in the end tests ran successfully and I hoped this was one time thing. 

Turns out that was not the case at all!

We are a small team which can't afford to have full fledged test suits just yet. We decided to be selective. First, we wanted to write a smoke test suite for integration testing. Secondly we want to write UI unit tests for important screens. These two type of tests required two different `TestRunners`. As evident from my [previous article](https://medium.com/swlh/espresso-unit-testing-and-things-to-watch-out-in-android-b5435a91c677) for UI unit tests we required a [custom runner](https://github.com/abhishekBansal/mvvm-architecture-demo/blob/master/app/src/androidTest/java/com/abhishek/mvvmdemo/MyTestRunner.kt) and [Test Application](https://github.com/abhishekBansal/mvvm-architecture-demo/blob/master/app/src/androidTest/java/com/abhishek/mvvmdemo/TestApplication.kt) but, for integration tests we need to invoke our original Application instance. This is where problems started.

`Instrumentation Test Runner` need to be specified in gradle file inside `build.gradle` of module under the test.

```groovy
testInstrumentationRunner "androidx.test.runner.AndroidJUnitRunner"
```

First thing that we came up was to add two different `Product Flavors` in our app. Different flavors allowed specifying different test runners like this.

```groovy
flavorDimensions 'defaultDimension'
productFlavors {
    integrationTest {
        testInstrumentationRunner "androidx.test.runner.AndroidJUnitRunner"
        dimension 'defaultDimension'
    }

    normal {
        testInstrumentationRunner "com.abhishel.mvvmdemo.MyTestRunner"
        dimension 'defaultDimension'
    }
}
```
While this worked it didn't feel like a right solution because 
1. First, different test suit isn't a different `productFlavor` as such 
2. More importantly, we already have four different `buildTypes` in our app. That makes it eight(2x4) different build variants of app in total. Existing 4 build variants are enough to create confusion at times, having eight felt like a total overkill.

Although some people were suggesting [command line](https://stackoverflow.com/a/38676843/1107755) based solutions, I really wanted to be able to just hit `Run` button in `Android Studio` and run my tests(too much to ask?). I then stumbled upon Google's official [Android Testing Blueprints](https://github.com/googlesamples/android-testing-templates/tree/master/AndroidTestingBlueprint-kotlinApp) and voila! this seemed like a perfect solution to all my problems.

As demonstrated in these blueprints there exists an `Android Testing Gradle Plugin` which allows one to write test in a completely separate gradle module. This module will `target` the module under test and will contain all your testing classes. This seems like a pretty clean way of doing it. I quickly refactored my testing code as per the blueprint.

To my surprise(or not so surprise :-/)I started getting many weird issues related to resources and drawables. Crashstacks would suggest that a drawable which is not even being used in screens under test is not actually a drawable. Sometimes it would even try to load a `dimen` value as drawable ?!?
I actually faced all different issues described by people in this [Appium github issue thread](https://github.com/appium/appium-espresso-driver/issues/449). Even after moving few libraries here and there restricting transitive dependencies from libraries as people were suggesting at different places didn't help.

After all this drama I began to suspect if there is something wrong with `AndroidX` setup because these blueprints [haven't been maintained in last two years at-least](https://github.com/googlesamples/android-testing-templates/tree/master/AndroidTestingBlueprint-kotlinApp). In order to debug the issue, first thing I tried was to fork official repo. I then updated dependencies, gradle and kotlin versions. Blueprint still worked fine :(. I [created a PR](https://github.com/googlesamples/android-testing-templates/pull/37) in a hope that somebody might just wake up and decide to give this repo a long due affection. 

When this debugging strategy did not work I started implementing this configuration in [one of my existing sample](https://github.com/abhishekBansal/mvvm-architecture-demo/tree/integration_testing_module) which has more modern Android developement elements like `Lifecycle`, `AppCompat` and `ViewModels` etc. This sample is very small compared to my original app but does a lot more than over simplified blueprint. Here I could confirm that this is an issue with `com.android.test` plugin and something is very wrong with the way resources are being compiled here. Upon spending a couple of more hours on internet I found [this bug](https://issuetracker.google.com/issues/129421689) which confirms my findings.

### Conclusion
I spent about 20 frustrating hours figuring all this out. While everyone(including folks at Google) talks a lot about testing, situation on the ground seems to be pretty difficult. As a next step for automation I am going to explore running tests from command line. For development purposes we will just replace `testInstrumentationRunner` value manually in our local branches. I will also be moving to feature development primarily and take this setup as a side project.

I am also open for suggestions at this point, if you think I am misinformed or I have wrongly presented something please feel free to correct me.

Happy Coding!