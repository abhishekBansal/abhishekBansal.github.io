---
layout: post
title: Fast migration from Kotlin Synthetics to View Binding- Tips and Tricks
date:   2021-01-29 8:00:00
author: Abhishek
categories: Android, Gradle, View Binding, Synthetics, Kotlin Extensions, Kotlin
img: "viewbinding/viewbinding_cover.jpg"
---

[Android Kotlin Extension Gradle](https://plugins.gradle.org/plugin/org.jetbrains.kotlin.android.extensions) plugin is [deprecated](https://android-developers.googleblog.com/2020/11/the-future-of-kotlin-android-extensions.html) in favor of [ViewBinding](https://developer.android.com/topic/libraries/view-binding). That marks the end of too good to be true `kotlinx.android.synthetic` bindings for XML files. While `synthetics` were very convenient they had the following shortcomings
1. They only worked in Kotlin and are not supported in Java
2. They pollute the global namespace, as in every view is available everywhere in your app.
3. They don't expose nullability information

All of the above is solved in the new `View Binding` library, however, in one of my projects, there is heavy use of synthetics. In absence of an automated migration tool, it is a big pain to move every Activity, Fragment, View, Adapter, etc. to the new method of accessing views in a relatively large codebase. Also, accessing views consist of a sizable chunk of code in any app. It's not a good idea to rely too long on deprecated methods for something this important. In this article, I am going to share a few tips and tricks that can help you complete this otherwise cumbersome migration in a faster and easier way.

## Use viewBinding delegate
Using `ViewBinding` in a `Fragment` requires you to do the following.

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
That's lot less code. `by viewBinding()` [is a custom property delegate](https://medium.com/@Zhuinden/simple-one-liner-viewbinding-in-fragments-and-activities-with-kotlin-961430c6c07c) by [Gabor Varadi](https://medium.com/@Zhuinden). Apart from syntactic sugar it also has added benefit that you don't accidentally leave setting your `binding` instance to null in `onDestroyView`. Why you need to do that is explained in [official docs here](https://developer.android.com/topic/libraries/view-binding#fragments). I encourage you to read his post and understand how he has implemented this and how to use this in your Activities.

## Consistent naming for ViewBinding instance
While basic it is an important step before we move on. I recommend that you name your `View Binding` instance consistently(in fact use the same name wherever possible). In the above example I have named it `binding` instead of `demoBinding` or `demoFragmentBinding` etc. I always name my `View Binding` instance `binding` in all fragments and activities. You can choose this name as you like but keep it the same everywhere. This is to make live templates work that we will see next.


## Live Template for property delegate
We still have to go type this in every fragment/activity that we have
```kotlin
private val binding by viewBinding(FragmentDemoBinding::bind)
```

You could copy-paste this line every-time but it is faster to use the `Android Studio Live Templates` feature. New live templates can be created by going to `Preferences -> Editor -> Live Templates`. I usually create a new template group for project-specific templates. Create a new template group by pressing `+` on the right side of the preference dialog.

![Creating a new Live Template group in Android Studio](/assets/images/viewbinding/new_group_live_template.png)

Now create a new template by pressing `+` again or copy an existing template to the newly created group. I usually copy an existing template but for purpose of this article, I'll show how to create one from scratch. Next, you need to give an `abbreviation` for your new template. This is what you will be typing in the editor to access the template. I will name it `byVB` which is shortened form of `by viewBinding`. Put appropriate description so that other people in your team can understand what this template is about if you choose to share it. In `Template text` section put the following code 
```kotlin
private val binding by viewBinding($CLASS$::$METHOD$)
```
This is what will automatically be typed when you select the template from suggestions in the editor. The last bit is to select context, context lets Android Studio know where to apply this template. Select `Kotlin->Class` for purpose of this template. Here is how final this looks

![Creating a live template for View Binding instance declaration in Android Studio](/assets/images/viewbinding/binding_object_declaration_live_template.png)

Let us see this in action

![Live template for View Binding instance declaration in Android Studio](/assets/images/viewbinding/binding_declaration.gif)

Isn't that cool? That's just a declaration of binding instance though. We still need to make changes in code such that all views are accessed via this binding instance. Let's see how we can minimize that effort.

## Use Kotlin scope functions where applicable
We declared binding instance quickly but what about its actual usage? We still need to access views like `binding.textView` and so on everywhere. 
Let's say you have this bunch of code in your Activity/Fragment somewhere

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
`with` is one of the scope functions that language provides us. You might also need to use `binding?.apply {}` at times. Read more about scope functions [here](https://kotlinlang.org/docs/reference/scope-functions.html) and [here](https://blog.mindorks.com/using-scoped-functions-in-kotlin-let-run-with-also-apply)

If you are thinking that its too much work to type in that `with` and then move existing code inside `{}` block, stop thinking and read on.

## Use Live Template to insert scope functions

If you have worked on Android Studio for some time you know that you can select a bunch of code and then press CMD+OPT+T (ALT+CTRL+T on windows) on mac and open a context menu that lets you surround that selected code with blocks like `if`, `if/else`, `try/catch` etc. We are gonna create a similar template for view binding with block. This template needs to go in a special template group called `surround`. Since we have already gone through the process of creating a new Live Template I will just show important bits here. Here is the live template code that we need to put
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

Here is the final configuration of this template

![Live template for Scope function](/assets/images/viewbinding/surround_live_template.png)

Here it is in action

![Wrapping a code block in scope function](/assets/images/viewbinding/binding_scope.gif)

That's it, folks. This workflow helped me in boosting the migration speed by 2x to 3x. I hope you can gain some speedups in your workflow too. If you have a trick up your sleeve as well then do share.

Happy Coding!