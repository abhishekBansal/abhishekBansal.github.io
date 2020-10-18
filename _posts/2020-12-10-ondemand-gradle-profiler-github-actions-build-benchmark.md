---
layout: post
title:  "Automated, on demand benchmarking of Android Gradle builds with Github Actions"
date:   2020-10-12 8:00:00
author: Abhishek
categories: Android, Gradle, Android Build, Benchmarking, Gradle Profiler, Performance
---

Slow build speeds are not something which are unfamiliar to us Android Developers. It specifically becomes a nuisance when your code base is large. Now a days `Kotlin` is preferred choice of development language for many developers(including myself). Since `Kotlin` is [known to have larger build times](https://medium.com/keepsafe-engineering/kotlin-vs-java-compilation-speed-e6c174b39b5d) then `Java` it becomes even more important keep a tab on those build times.

There are [many](https://www.freecodecamp.org/news/how-to-improve-the-build-speed-of-your-android-projects-bd49029d8602/) [build performance](https://developer.android.com/studio/build/optimize-your-build) [optimizations techniques and tutorials](https://medium.com/androiddevelopers/improving-build-speed-in-android-studio-3e1425274837) are available out there that one can apply to keep build speeds in check. However, one also need to measure efficacy of those optimizations. Benchmarking build after every change can be tedious and time consuming task, again, specially for larger code bases. Thats where [gradle-profiler](https://github.com/gradle/gradle-profiler) and [Github Actions](https://github.com/features/actions) come to our rescue.

In this article I will explain how I built an `automated`, `on demand` build benchmarking workflow using `Github Actions`. I specifically chose to make workflow `on demand` and not `continuous` because it is un-necessary and suboptimal from costing and carbon perspective. Ideally you would want to check build times when you have introduced a new library, or changed some gradle configuration or version, or did some refactor effort for modularisation etc. It can also be determined at the time of code review whether a change set can affect build speeds. That's just my opinion though, I will also show how to run benchmarking on continuous basis as well. Let's begin.

### About Github Actions
[A lot](https://medium.com/@wkrzywiec/github-actions-for-android-first-approach-f616c24aa0f9) [has already been written](https://proandroiddev.com/how-to-github-actions-building-your-android-app-773779bcacab) [about Github Actions integration with Android builds](https://www.coletiv.com/blog/android-github-actions-setup/), but, for the sake of completeness I would like to put my interpretation in simple words. [Github actions](https://github.com/features/actions) essentially provide you command line access of a fresh computer with operating system of your choice. You can program this computer via [`YML`](https://yaml.org/) based workflow script. You can download your code on this computer, do bunch of stuff with it and then, you can also upload results back to your `Github` repository. You can even send them to your server via APIs and upload to your Slack team. Possibilities are limitless when you have a whole machine to fiddle with :). What I like about this approach is reproducibility, because, every-time your workflow runs it starts on a fresh instance, hence environment always remains constant.

One such complete workflow is called an Action which [can be redistributed](https://docs.github.com/en/free-pro-team@latest/actions/creating-actions) for other's to use. Pretty cool! Github actions are event driven which off-course integrate very well Github events like pull request, issue creation, comment, push etc. We will use this integration when we build our own action to benchmark builds.

### About Gradle Profiler
[Gradle Profiler](https://github.com/gradle/gradle-profiler) is a great tool for automating profiling and benchmarking information of gradle builds. It takes care of consistency and reproducibility of builds by running warmup builds before running actual measurement builds. `gradle-profiler` can be used to benchmark any gradle task, we will be using it to measure clean debug builds. There are two ways of using `gradle-profiler`.
First directly from command line like 
> gradle-profiler --benchmark --project-dir <root-dir-of-build> <task>

Second 
> gradle-profiler --benchmark --scenario-file build_performance.scenarios --warmups 5 --iteration 10

Second is more convenient and more suitable for advanced scenarios. [`build_performance.scenarios`](https://github.com/gradle/gradle-profiler#advanced-profiling-scenarios) file is a configuration file in which you can write different scenarios to benchmark. Here is a simple config that we will be using for purposes of this article
```
# Can specify scenarios to use when none are specified on the command line
default-scenarios = ["assemble"]

# Scenarios are run in alphabetical order
assemble {
    # Show a slightly more human-readable title in reports
    title = "Clean Build Debug"
    # Run the 'assemble' task
    tasks = ["assembleDebug"]
    cleanup-tasks = ["clean"]
}
```
Contents of this file are self explanatory, we have essentially written `./gradlew clean assembleDebug` command in this scenario file.

### Lets Begin
Let's begin writing Github action workflow now. We are going to create two jobs, one for merge commit and one for PR head commit. Here is exactly what we are going to do
#### Job 1
1. Setup Trigger
2. Clone Repo with merge commit
3. Install JDK, SDKMAN and gradle-profiler
4. Run profiler
5. Save results for later comparison and use

#### Job 2
6. Repeat Step 2, 3 and 4 for PR Head commit this time
7. Download results from Step #5 and run a Python Script to compare both
8. Send results to Slack to your database or do whatever with it

### Step 1- Setup Action Trigger
Since we want to run this action on demand we will be using [Pull Request Comment Trigger](https://github.com/Khan/pull-request-comment-trigger) action to listen for Pull Request comments. As soon as trigger comment is posted action will start executing. I am using `benchmark-build` as trigger phrase for this workflow. This action requires following `YML` block in trigger section of the workflow
```yml
on:
  issue_comment:
    types: [created]
```

After that we can start writing our first Job
```yml
jobs:
  build-merge:
    runs-on: ubuntu-latest

    steps:
      - uses: khan/pull-request-comment-trigger@master
        id: check
        with:
          trigger: 'benchmark-build'
```

### Step 2- Clone repo with merge commit
We can use standard [checkout action](https://github.com/actions/checkout) with default values for cloning repo.
```yml
      - name: Clone Repo
        uses: actions/checkout@v2
        with:
          submodules: recursive
```
I have added `submodules: recursive` just in case your repo contains any submodules like one of my projects.

### Step 3- Install Required Dependencies
We need `JDK 1.8`, [`SDKMAN`](https://sdkman.io/) and [`gradle-profiler`](https://github.com/gradle/gradle-profiler) to build and benchmark Android build. `SDKMAN` is required to install `gradle-profiler`. Here are [detailed instructions](https://github.com/gradle/gradle-profiler#installing) on it. The `YML` block looks like this. `run` is used to execute commands on a terminal.

```yml
# Setup JDK on container
      - name: Set up JDK 1.8
        uses: actions/setup-java@v1
        with:
          java-version: 1.8
          
      - name: Install SDKMAN, Gradle Profiler
        run: |
         curl -s "https://get.sdkman.io" | bash
         source "$HOME/.sdkman/bin/sdkman-init.sh"
         sdk install gradleprofiler
```

### Step 4- Run Profiler
Somehow, I observed that `gradle-profiler` installation done in Step 3 was not available to next step so I fired benchmarking in same step only. That step now becomes
```yml
      - name: Install SDKMAN, Gradle Profiler and Begin Profiling
        run: |
         curl -s "https://get.sdkman.io" | bash
         source "$HOME/.sdkman/bin/sdkman-init.sh"
         sdk install gradleprofiler
         gradle-profiler --benchmark --scenario-file build_performance.scenarios --warmups 1 --iteration 1
```
*I have kept `--warmups` and `--iteration` values to 1 in order to do quick testing and prototyping. Please modify it as you see fit for your use case.*

### Step 5- Upload Result to Repo
After benchmarking is done we will be uploading result from docker container to Github repo, so that, we can download and use it later. Result is stored in a folder named `profile-out`. Here is how another standard action [`upload-artefact`](https://github.com/actions/upload-artifact) can be used to do this
```yml
      - uses: actions/upload-artifact@v2
        with:
         name: merge-benchmark
         path: profile-out/benchmark.csv
```

### Step 6- Run another job for head commit
By default if you specify multiple jobs in a workflow file they are started in parallel. For this case I wanted second job to start only after first job has finished running. This is because 
1. There is no point in running second job if first fails. 
2. In async jobs its difficult to determine which finished when and then create a third job for running analysis on benchmark results. 
3. This will also save us some execution minutes and cost in turn in case of failure.

Here is how its configured
```yml
  build-head:
    needs: build-merge
    runs-on: ubuntu-latest

    steps:
      # Clone repo
      - name: Clone Repo
        uses: actions/checkout@v2
        with:
          submodules: recursive
          ref: ${{ github.event.pull_request.head.sha }}
```

There are two things to notice here. First, `needs: build-merge` tells actions that this job is dependent on job with id `build-merge`. Only when that job is a success this job will begin its execution. Second, `ref: ${{ github.event.pull_request.head.sha }}` tells checkout action to pull PR head and not merge commit. After this we need to repeat installation steps as previous job.

```yml
      # Setup JDK on container
      - name: Set up JDK 1.8
        uses: actions/setup-java@v1
        with:
          java-version: 1.8

      - name: Install SDKMAN, Gradle Profiler and Begin Profiling
        run: |
          curl -s "https://get.sdkman.io" | bash
          source "$HOME/.sdkman/bin/sdkman-init.sh"
          sdk install gradleprofiler
          gradle-profiler --benchmark --scenario-file build_performance.scenarios --warmups 1 --iteration 1

      - uses: actions/upload-artifact@v2
        name: Archive Benchmark Result File
        with:
          name: head-benchmark
          path: profile-out/benchmark.csv
```

### Step 7: Download Previous result and Compare
We can download file that we previously uploaded via another action called [`download-artefact`](https://github.com/actions/download-artifact). I am going to download that in a folder called `profile-out-merge` in following way
```yml
      - uses: actions/download-artifact@v2
        with:
          name: merge-benchmark
          path: profile-out-merge
```
`merge-benchmark` is name that we gave this artefact in Step 5. Once we have both benchmark results on file system you can write simple scripts in language of your choice and do any processing per requirement. Output file is a simple CSV file which looks like this 
![gradle-profiler output](/assets/images/gradle_profiler_benchmark.png)

Here is a python script which simply takes two benchmark files and prints out mean execution time of build. This script also writes results in a file because it can then be used to send these results back to a `Slack` channel.
```python
import csv

def getResult(fileName):
    with open(fileName) as f:
        next(f)  # Skip the header
        reader = csv.reader(f, skipinitialspace=True)
        return dict(reader)

headResult = getResult('profile-out/benchmark.csv')
mergeResult = getResult('profile-out-merge/benchmark.csv')

headMean = headResult['mean']
mergeMean = mergeResult['mean']
buildStr = "Branch Head Build Time: " + headMean + " | Base Branch Build Time: " + mergeMean
# print result on console and write in a file
print buildStr
with open("benchmark-result.txt") as f:
  f.write(buildStr)
```

## See it in Action

You can see [the script](https://github.com/abhishekBansal/android-build-benchmark-github-actions/blob/master/.github/workflows/PRBuildBenchmark.yml) and try it out for yourself by forking [this repo](https://github.com/abhishekBansal/android-build-benchmark-github-actions). Here are a few screenshots for reference

1. Comment `benchmark-build` on PR
![](/assets/images/comment_on_pr.png)

2. Github Action triggered
![](/assets/images/action_triggred_by_comment.png)
