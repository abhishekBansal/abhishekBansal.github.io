---
layout: post
title: Controlling swipe direction with ViewPager2
date:   2021-08-17 8:00:00
author: Abhishek Bansal
categories: Android, ViewPager2, UI, Views
img: "viewbinding/viewbinding_cover.jpg"
---

The new [ViewPager2] api is replacement for good old [ViewPager] API that was used to create swipable views in Android. `ViewPager2` has lot of benefits over its predecessor like support for vertical swiping and dynamic number of fragments etc. Its also the recommended way creating swipeable views on Android moving forward.

In one of the places where we are using [ViewPager] in our app we enable/disable user swipe in either(left/right) or both directions at runtime. This is done on basis of some user actions. In `ViewPager` we did this by sub-classing `ViewPager` class and by overriding `onTouchEvent` and `onInterceptTouchEvent`. As [many] [of the] [stackoverflow] answers suggest it is possible to determine the direction in which user is attempting then swipe and then simply discarding those touch events.

We wanted to migrate this screen to `ViewPager2`, however, it is a `final` class which cannot be subclassed. This was a blocker for our migration till the time we found following solution.

Before jumping on to the solution it important to understand that unlike `ViewPager`, `ViewPager2` is essentially super customized `RecyclerView`. It uses `RecyclerView`'s super powers to render and reuse views. Infact, [a lot of the code was removed] from `ViewPager` and all that responsibility was given to  `RecyclerView` instead. 

Now that we know that `RecyclerView` handles view rendering, we can use `RecyclerView`'s APIs to handle swiping. RecyclerView takes `ItemTouchListener` which allows intercepting touch events. Let's first create an enum to specify all possible swipe direction combinations

```kotlin
enum class SwipeDirection {
    ALL, // swipe allowed in left and right both directions
    LEFT, // swipe allowed in only Left direction
    RIGHT, // only right
    NONE // swipe is disabled completely
}
```

This enum allows us to specify directions in which a user is allowed to swipe at any given point of time. Now we need to create a function which takes `MotionEvent` object and attempts to detect the direction user is trying to swipe, compare it with currently set `SwipeDirection` and then return `true` if user's intention and current setting matches `false` otherwise. Take a look at following function

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

For most part this is self explanatory but, most interesting bit is that we are saving current `x` coordinate when a `ACTTION_DOWN` event is received. That is, user has touced the screen to begin swipe. After that, when use begins the swipe an `ACTION_MOVE` event is received.Here we can take the diff of initial x position and current x position and figure out in which direction user is trying to swipe, compare it with current setting and take decision accordingly. I have left out other details of `ItemTouchListener` for brevity. You can find [full implementation here]().

Once this touch listener is set on VP2's RecyclerView, changing swipe control is matter of just setting the swipe direction on this touch listener as follows.
```kotlin
swipeControlTouchListener.setSwipeDirection(SwipeDirection.LEFT)
```

Now comes the dirty part where we have resort to a not so perfect solution and get access to `ViewPager2` to RecyclerView in a questionable way.
```kotlin
// apply touch listener on ViewPager RecyclerView
val recyclerView = binding.viewPager[0] as? RecyclerView
if (recyclerView != null) {
    recyclerView.addOnItemTouchListener(swipeControlTouchListener)
} else {
    Log.w(localClassName, "RecyclerView is null, API/Version changed ?!")
}
```
Here we are relying on `RecyclerView` being first child of VP2 always, one solution could be to use `findViewById` but thats not possible because id of this RecyclerView is generated at runtime using `ViewCompat.generateViewId()` as you should be able to notice in VP2 source code. If you got a better idea do let me know in comments.

Happy Coding!