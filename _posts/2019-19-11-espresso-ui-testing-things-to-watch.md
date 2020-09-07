---
layout: post
title:  "Espresso unit testing and things to watch out in Android"
date:   2019-11-19 8:00:00
author: Abhishek
categories: Android, Testing, UI Testing, Espresso
img: espresso-testing-cover.jpeg
---

In my brief experience with Android developers and community so far I have seen lot of people talking about automated testing and its importance, but a very few people get the chance to actually write extensive tests for their code. They hardly ever have time for it. I also happen to be one of them. Although I have heard, read and talked about it a lot, I have very limited hands on experience. 

In one of the apps I am currently working on, team finally decided to start writing tests for good. We were first aiming to build a smoke testsuite for complete app so we started with UI testing with [Espresso](https://developer.android.com/training/testing/espresso). We also use [Koin](http://insert-koin.io/) as DI framework in this project.

Honestly, I found that testing on Android is a mess for beginners at-least. In this article I am going to point out some of the pitfalls and mistakes that we came across.

## Libraries
There are literally tons and tons of google libraries available which are available for testing. Official [AndroidX testing guide](https://developer.android.com/training/testing/set-up-project) lists more then 10 libraries for doing just UI tests. These are the ones that you add as `androidTestImplementation` in your `build.gradle` similary there are other bunch of libraries for JVM unit tests. Not just these, in lot of tutorials, code samples and stackoverflow posts you will find older non AndroidX version of these. 

I would recommend to not include all these at once in your project. Just start with one and then include the ones that you need, so that, you exactly know what you are including and for what. This just helps in building understanding step by step.

I required following libs in order to make my bare minimum sample work

```groovy
    // testing with Koin
    androidTestImplementation ("org.koin:koin-test:$koin_version") {
        exclude group: 'org.mockito'
    }

    androidTestImplementation 'androidx.test.espresso:espresso-core:3.2.0'
    // stuff like ActivityTestRule
    androidTestImplementation 'androidx.test:rules:1.2.0'
    // AndroidJUnit4
    androidTestImplementation 'androidx.test.ext:junit:1.1.1'
    // mockito android
    androidTestImplementation 'org.mockito:mockito-android:3.1.0'
```

## Test/Mock/Fake Application
You need a mock application class as an entry point for UI tests. You can initialize your dependency graph here. Instead of your actual `Application` class this will instantiated with your tests. You will also need a custom `Runner` instance where you will instantiate Application.
Here is my `TestApplication`
```kotlin
class TestApplication : Application() {
    override fun onCreate() {
        super.onCreate()

        startKoin {
            androidLogger()
            androidContext(this@TestApplication)
            // no modules are loaded to begin with
            // modules will be loaded in respective tests as per requirement
            modules(emptyList())
        }
    }
}
```

Here is my `TestRunner`
```kotlin
class MyTestRunner : AndroidJUnitRunner() {
    override fun newApplication(cl: ClassLoader?, className: String?, context: Context?): Application {
        return super.newApplication(cl, TestApplication::class.java.name, context)
    }
}
```

You will then also need to specify this test runner in your `build.gradle` `defaultConfig` block
```groovy
testInstrumentationRunner "com.abhishek.mvvmdemo.MyTestRunner"
```

## Mockk vs Mockito
I started with [Mockito](https://site.mockito.org/) for mocking dependencies. Then A few people recommended [Mockk](https://mockk.io/#dsl-examples) over `Mockito`. I tried it and I liked it intially. One of the great thing about it was that it doesn't require you to declare your `Kotlin` classes and methods as `open`. Also I found that there were fewer issues to deal with when using `Mockk` then with `Mockito`.

But then I tested my app on API 27, tests that were running fine on API 28 and 29 suddenly started breaking. Turns out `Mockk` does that `open` magic only for devices with API level >= 28. This was a deal breaker!

At this point we decided to go back to `Mockito` as its a established library, some of the test libraries like `koin-test` were already using it under the hood and we will have one less library to deal with.

With our limited knowledge of `Mockk` this is not a final decision. If in future we feel that Mockk is more suitable option for Kotlin then we may decide to migrate.

### @OpenForTesting to Rescue
[Yigit Boyar](https://github.com/yigit) wrote this very useful [OpenForTesting](https://github.com/android/architecture-components-samples/issues/346) annotation which can be used to make all classes and methods open just for testing. Here is a great [step by step tutorial](https://proandroiddev.com/mocking-androidtest-in-kotlin-51f0a603d500) on how to set this up in your project. This worked great and now our tests started passing on API < 28.

Remember even if any of your class is `open` in your codebase you still need to annotate it with `@OpenForTesting`(or whatever you choose to name it) in order to be able to mock its methods. Remember, that in `Kotlin` all methods in a class are closed by default. We spent hours on this because somehow, we managed to ignore this fact.

## ActivityTestRule
Test rules are great way of reusing code across your tests. Here is a [great article](https://medium.com/@elye.project/all-about-testrule-a-steroid-before-after-a74ef421e3e5) that explains test rules in detail. [ActivityTestRule](https://developer.android.com/reference/android/support/test/rule/ActivityTestRule) is one such test rule provided by good folks at Android. It takes care of managing an `Activity` environment for your test. 

My bad that I didn't read/interpret documents properly and got an issue where I wasted a full day. My problem was that this rule was launching activity even before I loaded my dependencies with `Koin` and, as a result was getting `NoBeanDefFoundException`. More on this issue on this [Stackoverflow post](https://stackoverflow.com/questions/58728051/nobeandeffoundexception-with-mock-viewmodel-testing-with-koin-espresso/). (Special thanks to [basilisk](https://stackoverflow.com/users/493321/basilisk) for solution :))

## Exceptions!
All was good till this point and now my dependencies were getting loaded properly. I went on and wrote my first line in `@Test` method. 
```kotlin
// nothing to test lets just try to write my name in an editText
onView(withId(R.id.emailEt))
    .perform(ViewActions.typeText("Hello World!"))
```

You would think what can go wrong with that, right? Well first exception that I encountered here was 
> PerformException: Error performing 'type text(Hello World!)' on view 'Animations or transitions are enabled on the target device.

As per official [Espresso docs](https://developer.android.com/training/testing/espresso/setup) device animations should be turned off. More on this exception in next section.

After I turned off the animations I fired up my test again only to find this exception
```
Caused by: androidx.test.espresso.InjectEventSecurityException: java.lang.SecurityException: Injecting to another application requires INJECT_EVENTS permission
```
Well, I just tried to write a string in a `EditText` didn't I? Turns out that Android 10 running on my phone tries to autofill the given editText. Now this auto-fill is important from user experience perspective and I cannot disable it in XML. Thanks to `ActivityRule` I was able to do this
```kotlin
activityRule.activity.findViewById<EditText>(R.id.emailEt)
?.importantForAutofill = View.IMPORTANT_FOR_AUTOFILL_NO
```

### Random Issues ?!?
Most surprising thing about this endeavour were random issues, some of these issues appear and disappear without even a code change. Cleaning the project, connecting disconnecting device seem to affect these but there is no clear pattern.

For example I was almost always getting an exception complaining that `ViewModel.clear()` method wasn't mocked or definition not found. This method is final and cannot be mocked. Workaround is to add `ActivityRule.finishActivity()` call in `@After` method. Surprisingly, I am not able to reproduce this at the time of writing.

There are time when a tests executes twice, it passes first time and then fails the second time. On `Stackoverflow` some people were suggesting that this might be happening because test are being run once as a part of suite and then individually. I am not fully convinced here, however, cleaning the build and re-running the test seem to resolve this issue. 

## Turn off Animation

Espresso tests require that device animations should be turned off. However, I have observed that if you pass `initialTouchMode=true` in `ActivityTestRule` this is no more a requirement.

There is a gradle flag which allows you to disable device animations.
```groovy
android {
    testOptions {
        animationsDisabled = true
    }
}
```

Well, [this doesn't work](https://stackoverflow.com/questions/43474144/what-does-the-testoptions-animationsdisabled-property-in-android-gradle-plugin-d) or not reliably in my experience at the very least. Don't waste your time on this like I did.

As of now we have started writing our tests without disabling them. But, if this a problem in future we can consider deploying some [custom solution like this](https://proandroiddev.com/one-rule-to-disable-them-all-d387da440318).


As you can see it was pretty daunting to setup UI unit testing in our Android app. I hope this was one time annoyance. I have tried to compile all the issues that me and my team faced so that you as a reader know what you are getting into. Here is the [source code](https://github.com/abhishekBansal/mvvm-architecture-demo) of project I was working with. 

Please feel free to drop your feedback and suggestions.

Happy Coding!

Note: This article was *Originally published at [Espresso unit testing and things to watch out in Android](https://medium.com/swlh/espresso-unit-testing-and-things-to-watch-out-in-android-b5435a91c677)*