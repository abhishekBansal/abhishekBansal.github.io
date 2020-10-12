---
layout: post
title:  "Automated, on demand benchmarking of Android Gradle builds with Github Actions"
date:   2020-10-12 8:00:00
author: Abhishek
categories: Android, Gradle, Android Build, Benchmarking, Gradle Profiler, Performance
---

Slow build speeds are not something which is unfamiliar to us Android Developers. It specifically becomes a nuisance when your code base is large. Now a days `Kotlin` is preferred choice of development language for many developers(including myself). Since `Kotlin` is [known to have larger build times](https://medium.com/keepsafe-engineering/kotlin-vs-java-compilation-speed-e6c174b39b5d) then `Java` it becomes even more important keep a tab on those build times.

There are many build performance optimizations techniques and tutorials are available out there that one can apply to keep build speeds in check. However, one also need to measure efficacy of those optimizations. Benchmarking build after every change can be tedious and time consuming task again specially for larger code bases. Thats where [gradle-profiler](https://github.com/gradle/gradle-profiler) and [Github Actions](https://github.com/features/actions) come to our rescue.

In this article I will explain how I built an `automated`, `on demand` build benchmarking workflow using `Github Actions`. I specifically chose to make workflow `on demand` and not `continuous` because it is un-necessary and suboptimal from costing and carbon perspective. Ideally you would want to check build times when you have introduced a new library, or changed some gradle configuration or did some refactor effort for modularisation etc. Or it can also be determined at the time of code review whether a change set can affect build speeds. That's just my opinion though, I will also show how to run benchmarking on continuous basis as well. Let's begin.

### About Github Actions
A lot has already been written about Github Actions integration with Android builds, but, for the sake of completeness I will like put my interpretation. [Github actions](https://github.com/features/actions) essentially provide you command line access of a fresh computer with operating system of your choice. You can program this computer via [`YML`](https://yaml.org/) based workflow script. You can download your code on this computer do bunch of stuff with it and then you can also upload results back to your `Github` repository or send them to your server via APIs or upload to your Slack team. Possibilities are limitless when you have a whole machine to fiddle with :). What I like about this approach is reproducibility, because, every-time your workflow runs it starts on a fresh instance, hence environment always remains constant.

One such complete workflow is called an Action which [can be redistributed](https://docs.github.com/en/free-pro-team@latest/actions/creating-actions) for other's to use. Pretty cool! Github actions are event driven which off-course integrate very well Github events like pull request, issue creation, comment, push etc. We will use this integration when we build our own action to benchmark builds.

### About Gradle Profiler
[Gradle Profiler](https://github.com/gradle/gradle-profiler) is a great tool for automating profiling and benchmarking information of gradle builds. It takes care of consistency and reproducibility of builds
