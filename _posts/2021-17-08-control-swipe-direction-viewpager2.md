---
layout: post
title: Controlling swipe direction with ViewPager2
date:   2021-08-17 8:00:00
author: Abhishek Bansal
categories: Android, ViewPager2, UI, Views
img: "viewbinding/viewbinding_cover.jpg"
---

The new [ViewPager2] api is a replacement for good old [ViewPager] API that was used to create swipable views in Android. `ViewPager2` has lot of benefits over its predecessor like support for vertical swiping and dynamic number of fragments etc. Its also the recommended way creating swipeable views on Android moving forward.

In one of the places where we are using [ViewPager] in our app we enable/disable user swipe at runtime in either(left/right) or both directions. This is done on basis of different user input. In `ViewPager` we did this by sub-classing `ViewPager` class and then overriding `onTouchEvent` and `onInterceptTouchEvent`. As [many] [of the] [stackoverflow] answers suggest it is possible to determine the direction of swipe that user is attempting and then simply discard those touch events.

We wanted to migrate this screen to `ViewPager2`. However, it is a `final` class which cannot be subclassed. This was a blocker for our migration till the time we figured a solution.

Before jumping on to the solution it important to understand that unlike `ViewPager`, `ViewPager2` is essentially heavily customized `RecyclerView`. It uses `RecyclerView`'s super powers to render and reuse views. Infact, [a lot of the code was removed] from `ViewPager` and all that responsibility was given to  `RecyclerView` instead. 

Now that we know that `RecyclerView` handles view rendering, we can use `RecyclerView`'s APIs to intercept swiping. RecyclerView has a useful method `addOnItemTouchListener` which takes an object of `RecyclerView.OnItemTouchListener`. This allows us to intercept and process touch events on RecyclerView. 

We will create a subclass of `RecyclerView.OnItemTouchListener` and call that `SwipeControlTouchListener`. In order to control swipe direction at runtime lets first create an enum call to specify all possible swipe direction combinations.

```kotlin
enum class SwipeDirection {
    ALL, // swipe allowed in left and right both directions
    LEFT, // swipe allowed in only Left direction
    RIGHT, // only right
    NONE // swipe is disabled completely
}
```

This enum allows us to specify directions in which a user is allowed to swipe at any given point of time. Now we need to create a function which takes `MotionEvent` object and attempts to detect the direction user is trying to swipe, compare it with currently set `SwipeDirection` and then return `true` if user's intention and current setting matches and `false` otherwise. Here is that function

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

For most part this is self explanatory. Lets look at the most interesting bit. We are saving current `x` coordinate when a `ACTTION_DOWN` event is received. That is, user has touced the screen to begin swipe. After that, when use begins the swipe, an `ACTION_MOVE` event is received. Here we can take the diff of initial x position and current x position in order to figure out what direction user is trying to swipe to, then compare it with current setting and take decision accordingly. I have left out other details of `SwipeControlTouchListener` for brevity. You can find [full implementation here]().

Once this touch listener is set on VP2's RecyclerView, changing swipe control is matter of just setting the swipe direction on this touch listener as follows.
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
Here we are relying on `RecyclerView` being first child of VP2 always, one solution could be to use `findViewById` but thats not possible because id of this RecyclerView is generated at runtime using `ViewCompat.generateViewId()`. If you take a look at VP2's source code you will know what I mean. 

This is not a 100% perfect solution but a pretty good workaround for us for time being. This is also because I don't expect too many changes in ViewPager2's layout anytime soon, also it will use `RecyclerView` for forseeable future. There is one other solution discussed in this [stackoverflow thread]() and here is a [feature request]() that I filed on Google bug tracker for this.

If you think this can be improved let me know in comments.

Happy Coding!