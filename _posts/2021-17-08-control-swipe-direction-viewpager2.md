---
layout: post
title: Controlling swipe direction with ViewPager2
date:   2021-08-17 8:00:00
author: Abhishek Bansal
categories: Android, ViewPager2, UI, Views
img: "viewpager2/swipe-control.png"
---

The new [ViewPager2](https://developer.android.com/training/animation/vp2-migration) API is a replacement for the good old [ViewPager](https://developer.android.com/reference/kotlin/androidx/viewpager/widget/ViewPager) API that was used to create swipeable views in Android. `ViewPager2` has a lot of benefits over its predecessor like support for vertical swiping, the possibility of using the diff-util and the dynamic number of fragments, etc. It is also the recommended way of creating swipeable views on Android [moving forward](https://developer.android.com/training/animation/vp2-migration#benefits).

In one of the places where we are using `ViewPager` in our app, we enable/disable user swipe at runtime in either(left/right) or both directions. This is done on basis of different user input. In `ViewPager` we did this by sub-classing `ViewPager` class and then overriding `onTouchEvent` and `onInterceptTouchEvent`. As [many](https://stackoverflow.com/questions/24356023/android-viewpager-detect-swipe-direction) [of the](https://stackoverflow.com/questions/19602369/how-to-disable-viewpager-from-swiping-in-one-direction) [stackoverflow](https://stackoverflow.com/questions/18660011/viewpager-disable-swiping-to-a-certain-direction/18661040#18661040) answers suggest it is possible to determine the direction of swipe that user is attempting and then simply discard those touch events. 

We wanted to migrate this screen to `ViewPager2`. However, it is a [`final`](https://android.googlesource.com/platform/frameworks/support/+/refs/heads/androidx-collection-release/viewpager2/src/main/java/androidx/viewpager2/widget/ViewPager2.java?source=post_page---------------------------#63) class that cannot be subclassed. This was a blocker for our migration till the time we figured a solution.

Before jumping on to the solution it is important to understand that unlike `ViewPager`, `ViewPager2` is essentially a heavily customized `RecyclerView`. It uses `RecyclerView`'s superpowers to render and reuse views. In fact, [a lot of the code was removed](https://medium.com/google-developer-experts/exploring-the-view-pager-2-86dbce06ff71#3566) from `ViewPager` and all that responsibility was given to  `RecyclerView` instead. 

Now that we know that `RecyclerView` handles view rendering, we can use `RecyclerView`'s APIs to intercept swiping as well. RecyclerView has a useful method `addOnItemTouchListener` which takes an object of `RecyclerView.OnItemTouchListener`. This allows us to intercept and process touch events on RecyclerView. 

We will create a subclass of `RecyclerView.OnItemTouchListener` and call that `SwipeControlTouchListener`. To control swipe direction at runtime let's first create an enum call to specify all possible swipe direction combinations.

```kotlin
enum class SwipeDirection {
    ALL, // swipe allowed in left and right both directions
    LEFT, // swipe allowed in only Left direction
    RIGHT, // only right
    NONE // swipe is disabled completely
}
```

This enum enables us to specify directions in which a user is allowed to swipe at any given point in time. Now we need to create a function that takes `MotionEvent` object and attempts to detect the direction user is trying to swipe. It should then compare it with the currently set `SwipeDirection` and return `true` if the user's intention and current setting match or `false` otherwise. Here is that function

```kotlin
private fun isSwipeAllowed(event: MotionEvent): Boolean {
        if (direction === SwipeDirection.ALL) return true
        if (direction == SwipeDirection.NONE) //disable any swipe
            return false
        if (event.action == MotionEvent.ACTION_DOWN) {
            initialXValue = event.x
            return true
        }
        if (event.action == MotionEvent.ACTION_MOVE) {
            try {
                val diffX: Float = event.x - initialXValue
                if (diffX > 0 && direction == SwipeDirection.RIGHT) {
                    // swipe from left to right detected
                    return false
                } else if (diffX < 0 && direction == SwipeDirection.LEFT) {
                    // swipe from right to left detected
                    return false
                }
            } catch (exception: Exception) {
                exception.printStackTrace()
            }
        }
        return true
    }
```

For the most part, this is self-explanatory. Let us look at the most interesting bit. We are saving the current `x` coordinate when an `ACTION_DOWN` event is received. That is, the user has touched the screen to begin swipe. After that, when the user begins the swipe, an `ACTION_MOVE` event is received. Here we can take the diff between the initial x position and current x position to figure out what direction the user is trying to swipe to, then compare it with the current setting and take a decision accordingly. I have left out other details of `SwipeControlTouchListener` for brevity. You can find [full implementation here](https://github.com/abhishekBansal/SwipeControlViewPager2/blob/master/app/src/main/java/dev/abhishekbansal/swipecontrolviewpager2/SwipeControlTouchListener.kt).

Once this touch listener is set on VP2's RecyclerView, changing swipe control is a matter of just setting the swipe direction on this touch listener as follows.
```kotlin
swipeControlTouchListener.setSwipeDirection(SwipeDirection.LEFT)
```

Now here is a little hacky part where I had to get access to `ViewPager2` to RecyclerView in a questionable way.
```kotlin
// apply touch listener on ViewPager RecyclerView
val recyclerView = binding.viewPager[0] as? RecyclerView
if (recyclerView != null) {
    recyclerView.addOnItemTouchListener(swipeControlTouchListener)
} else {
    Log.w(localClassName, "RecyclerView is null, API/Version changed ?!")
}
```
Here we are relying on `RecyclerView` being the first child of VP2 which is an implementation detail and can change with library upgrades. One solution to this problem could be to use `findViewById` but, that's not possible because the id of this RecyclerView is generated at runtime using `ViewCompat.generateViewId()`. If you take a look at [VP2's source code](https://android.googlesource.com/platform/frameworks/support/+/refs/heads/androidx-collection-release/viewpager2/src/main/java/androidx/viewpager2/widget/ViewPager2.java?source=post_page---------------------------#146) you will know what I mean. 

This is not a 100% perfect solution but a pretty good workaround for us for time being. This is also because I don't expect too many changes in ViewPager2's layout anytime soon, also, it will be using `RecyclerView` for the forseeable future. There is one other solution discussed in this [stackoverflow thread](https://stackoverflow.com/questions/56647971/how-to-disable-swiping-in-specific-direction-in-viewpager2) and here is a [feature request](https://issuetracker.google.com/issues/195903733) that I filed on Google bug tracker for this.

If you think this can be improved let me know in the comments.

**Sample App with source:**
[SwipeControlViewPager2](https://github.com/abhishekBansal/SwipeControlViewPager2)

**Same result with Gesture Overlay:**
We also tried a [`GestureOverlay` based approach](https://medium.com/@ramt57/viewpager2-disable-swiping-in-a-specific-direction-317265b554ea) which works pretty well in most of the cases. We ran into some minor issues related to child view scrolling and fast swiping. In interest of time we finalised on approach discussed above. You can deep dive on GestureOverlay based [approach here](https://medium.com/@ramt57/viewpager2-disable-swiping-in-a-specific-direction-317265b554ea).

**Credit:**
The `SwipeControlTouchListener` is slightly modified version of this [stackoverflow answer](https://stackoverflow.com/questions/19602369/how-to-disable-viewpager-from-swiping-in-one-direction).

Happy Coding!