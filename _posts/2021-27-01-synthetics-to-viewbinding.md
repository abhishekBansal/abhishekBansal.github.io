---
layout: post
title:  "Automated, on-demand benchmarking of Android Gradle builds with Github Actions"
date:   2020-10-23 8:00:00
author: Abhishek
categories: Android, Gradle, Android Build, Benchmarking, Gradle Profiler, Performance
img: "build-benchmark/banner.jpg"
---

[Android Kotlin Extension Gradle](https://plugins.gradle.org/plugin/org.jetbrains.kotlin.android.extensions) plugin is [deprecated](https://android-developers.googleblog.com/2020/11/the-future-of-kotlin-android-extensions.html) in favor of [ViewBinding](https://developer.android.com/topic/libraries/view-binding). That marks the end of too good to be true `kotlinx.android.synthetic` bindings for XML files. While `synthetics` were really convenient they had following shortcomings
1. They only worked in Kotlin and are not supported in Java
2. They pollute the global namespace, as in every view is available everywhere in your app.
3. They don't expose nullability information

All of above is solved in new `View Binding` library, however, in one of my projects there is a heavy use of synthetics. In absence of an automated migration tool it is a big pain to move every Activity, Fragment, View, Adapter etc. to new method of accessing views in relatively large codebase. Also, accessing views consist of sizable chunk of code in any app. It's not a good idea to rely too long on deprecated methods for something this important. In this article I am going to share a few tips and tricks that can help you complete this otherwise cumbersome migration.

## Use viewBinding delegate
Using `ViewBinding` in a `Fragment` requires you to do following.

```kotlin
    class DemoFragment : Fragment() {
        // declare binding
        private var binding: FragmentDemoBinding? = null
        override fun onCreateView(inflater: LayoutInflater, container: ViewGroup?,
                                  savedInstanceState: Bundle?): View? {
            // inflate and return view
            binding = FragmentDemoBinding.inflate(inflater, container, false)
            return binding?.root
        }

        override fun onDestroyView() {
            super.onDestroyView()
            // make it null to handle different view lifecycle in fragments
            binding = null
        }
    }
```
While all you wanna do is this

```kotlin
class DemoFragment : Fragment(R.layout.fragment_demo) {
    private val binding by viewBinding(FragmentDemoBinding::bind)
}
```
That's lot less code. `by viewBinding()` [is a custom property delegate](https://medium.com/@Zhuinden/simple-one-liner-viewbinding-in-fragments-and-activities-with-kotlin-961430c6c07c) by [`Gabor Varadi`](https://medium.com/@Zhuinden). Apart from syntactic sugar it also has added benefit that you don't accidentally leave setting your `binding` instance to null in `onDestroyView`. Why you need to do that is explained in [official docs here](https://developer.android.com/topic/libraries/view-binding#fragments). I encourage you to read his post and understand how he has implemented this and how to use this in your Activities.

## Consistent naming for ViewBinding instance
While basic this is an important step before we move on. I recommend that you name your `View Binding` instance consistently(in fact use same name wherever possible). In above example I have named it `binding` instead of `demoBinding` or `demoFragmentBinding` etc. I always name my `View Binding` instance `binding` in all fragments and activities. You can choose this name as you like but keep it same everywhere. This is to make live templates work that we will see next.


## Live Template for property delegate
We still have to go type this in every fragment/activity that we have
```kotlin
    private val binding by viewBinding(FragmentDemoBinding::bind)
```

You could copy paste this line every-time but it can be done faster with help of Android Studio Live Templates feature. New live templates can be created by going to Preferences -> Editor -> Live Templates. I usually create a new template group for project specific templates. Create a new template group by pressing `+` on right side on preference dialog.

*** IMAGE FOR NEW GROUP ***

Now create a new template by pressing `+` again or copy an existing template to newly created group. I usually copy an existing template but for purpose of this article I ll show how to create one from scratch. Next you need to give an `abbreviation` for your new template. This is what you will be typing in editor to access the template. I will name it `byVB` which is shortened form of `by viewBinding`. Put appropriate description so that other people in your team can understand what this template is about if you choose to share it. In `Template text` section put following code 
```kotlin
    private val binding by viewBinding($CLASS$::$METHOD$)
```
This is what will automatically be typed when you select the template from suggestions in editor. Last bit is to select context, context let Android Studio know where to apply this template. Select `Kotlin->Object Declaration` for purpose of this template because we are really declaring an object here. Here is how final this looks

*** IMAGE FOR FINAL DECLARATION TEMPLATE ***

Lets see this in action

*** GIF FOR FINAL TEMPLATE IN ACTION ***

Isn't that cool? Still too much work to be done though, lets move on to next live template.

## Use Kotlin scope functions where applicable
We declared binding instance quickly but what about its actual usage? We still need to access views like `binding.textView` and so on everywhere. 
Lets say you have this bunch of code in your Activity/Fragment somewhere

```kotlin
    private fun showSuccessState(item: List<String>) {
        progressBar.setVisible(false)
        adapter.addItems(item)
        recyclerView.setVisible(true)
        successText.setVisible(true)
    }
```

one way to migrate this code is this

```kotlin
    private fun showSuccessState(item: List<String>) {
        binding.progressBar.setVisible(false)
        binding.adapter.addItems(item)
        binding.recyclerView.setVisible(true)
        binding.successText.setVisible(true)
    }
```

But thanks to Kotlin you can do this
```kotlin
    with(binding) {
        progressBar.setVisible(false)
        adapter.addItems(item)
        recyclerView.setVisible(true)
        successText.setVisible(true)
    }
```
`with` is one of scope functions that language provides us. You might also need to use `binding?.apply {}` at times. Read more about [scope functions here](https://kotlinlang.org/docs/reference/scope-functions.html) and [here](https://blog.mindorks.com/using-scoped-functions-in-kotlin-let-run-with-also-apply)

If you are thinking that its too much work to type in that `with` and then move existing code inside `{}` block, you think too much, you are in for a treat here.

## Use Live Template to insert scope functions

If you have worked on Android Studio for sometime you know that you can select bunch of code and then press CMD+OPT+T (ALT+CTRL+T on windows) on mac and open a context menu which lets you surround that selected code with blocks like `if`, `if/else`, `try/catch` etc. We are gonna create a similar template for view binding with block. This template needs to go in a special template group called `surround`. Since we have already gone through process of creating a new Live Template I will just show important bits here. Here is the live template code that we need to put
```kotlin
with(binding) { 
    $SELECTION$ 
}
```
similarly, you can create a template for `apply` case.
```kotlin
  binding?.apply { 
      $SELECTION$ 
  }
```

Here is final configuration of this template

*** IMAGE FOR FINAL WITH TEMPLATE ***

Here it is in ACTIOn

*** GIF ***