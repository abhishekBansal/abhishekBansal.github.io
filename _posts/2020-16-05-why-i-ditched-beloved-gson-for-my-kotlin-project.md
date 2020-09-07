---
layout: post
title:  "Why I ditched beloved Gson for my Kotlin project"
date:   2020-05-16 8:00:00
author: Abhishek
categories: Android, Gson, Moshi, Serialization, Deserialization
img: ditched-gson-moshi-cover.png
---

I have been developing Android apps for about 6.5 years now. For every small or big Android app I have churned out, I have blindly used `Gson` for serialization and deserialization. `Gson` seemed to be flawless as it almost read my mind.

This changed in one of my recent `Kotlin` only project. Like every other project we started off with Gson here as well. But then, things changed when we started facing dreaded `NullPointerException`s. But wait, did I not say `Kotlin`? And isn't Kotlin supposed to be our solution for this billion dollar problem?

Well no! Kotlin can only do so much if developers are so determined to get `NullPointerExceptions`. `Gson` uses reflection for deserializing `JSON` to `Java` or `Kotlin` objects. `Java` does not have notion of `nullable` types like `Kotlin`. `Gson` was originally written with Java in mind and it fails to understand difference between nullable and non-nullable types of Kotlin.

In this project our APIs were not mature enough to include all possible data validations and schema was not as strictly defined. So app will crash here and there while testing against dev servers, because some field that we have declared as non-nullable did not come in API or it did not come in right format. This happened because some other client is still developing feature or due to some bug. There was a possibility that something like this may happen in production as well.

In order to avoid this we were left with two choices, 

1. We make all our model fields `nullable`. i.e. our model will look something like this

    ```kotlin
    data class Person(
        val id: String?,
        val name: String?,
        val gender: Gender?,
        val address: Address?
    )
    ```
    Which feels wrong on so many levels. 
    1. This will make bunch of code wrapped in `.let{}` calls unnecessarily
    2. In my opinion it just looks ugly and makes code hard to follow because existence of every field is uncertain. 
    3. In most of the cases it does not even make sense to deserialize an object which is corrupted and unusable in app anyways. e.g. let's suppose there is a feature where app allows user to edit details and post it on server with help of `id`. In this case an object with `null` id will break the feature. Not to mention you will have to do special handling for this case. Its best to have id as `non-nullable` field.
2. Use some [evil getter setter hack like illustrated in this article](https://proandroiddev.com/most-elegant-way-of-using-Gson-kotlin-with-default-values-and-null-safety-b6216ac5328c)

Both of these are unacceptable and sort of break `Kotlin`'s soul. It is when I decided to move project to [Moshi](https://github.com/square/moshi). Moshi is small when compared to Gson and [tries to stay focused by doing less](https://github.com/square/moshi), but, its power lies in its simplicity. Moshi understands difference between `nullable` and `non-nullable` fields in Kotlin. This single handedly became deciding factor for me. 

Knowing that if a given field is declared as `non-null` it will not be null under any circumstance, suddenly increased the trust factor in my own code by manifolds. 

Let's look at some other benefits of using Moshi over Gson.

### Exception Handling
`Moshi` doesn't silently fail when there is a mismatch in field data type or nullbility contract. It raises a `JSONDataException` which clients are responsible for handling. This makes you aware about subtle mistakes which are easily missed with Gson.

### Moshi does not rely on Reflection
`Gson` is purely based on reflection where as `Moshi` supports both code generation and reflection. Reflection support can be added with `moshi-reflect` artefact but I don't really use it (Reflection is Slow and has same problems as Gson!). Code generation is supported in version 1.6+ with `moshi-kotlin-codegen` annotation processor.

### Moshi is Fast
As per [this benchmarking](https://zacsweers.github.io/json-serialization-benchmarking/) Moshi is faster then Gson in serialization and de-serialization.

## Let's see some code

Let's consider this JSON
```json
{
  "vehicles": [
    {
      "model": "m1",
      "type": "car",
      "tyres": [
        { "type": "mrf" },
        { "type": "apollo" }
      ]
    },
    {
      "type": "truck",
      "tyres": [
        { "type": "mrf" },
        { "type": "apollo" },
        { "type": "apollo" }
      ]
    }
  ]
}
```

Here corresponding Kotlin data classes

```kotlin
data class VehicleContainer(val vehicles: List<Vehicle>)

abstract class Vehicle {
    abstract val model: String
    abstract val type : String
    abstract val tyres: List<Tyre>?
}

data class Car(override val model: String,
               override val type: String,
               override val tyres: List<Tyre>?): Vehicle()

data class Truck(override val model: String,
                 override val type: String,
                 override val tyres: List<Tyre>?): Vehicle()

data class Tyre(val type: String)
```

Things to note:
1. We are working with polymorphic list of objects here. `Vehicle` is base class, `Car` and `Truck` are derived classes. 
2. `model` and `type` variables are mandatory and cannot be null.

Let's see how Gson and Moshi can deserialize this.

### Gson Deserialization
In order to deserialize `List<Vehicle>` we need to add support for [RuntimeTypeAdapterFactory](https://docs.mapbox.com/android/api/mapbox-java/libjava-geojson/4.6.0/com/google/Gson/typeadapters/RuntimeTypeAdapterFactory.html).

Here is how Gson object will be constructed
```kotlin
 val vehicleAdapterFactory: RuntimeTypeAdapterFactory<Vehicle> =
     RuntimeTypeAdapterFactory.of(Vehicle::class.java, "type")
       .registerSubtype(Vehicle::class.java, "car")
       .registerSubtype(Vehicle::class.java, "truck")

  val Gson = GsonBuilder().registerTypeAdapterFactory(vehicleAdapterFactory)
       .create()
```

Now to deserialize
```kotlin
val vehicleContainer = Gson.fromJson(jsonStr, VehicleContainer.class)
```

Note that even though our original JSON does not include `model` for `Truck` object Gson will deserialize this object and set `model` to `null` which may lead to unexpected `NullPointerException` later.

### Moshi Deserialization
Moshi also supports polymorphic serialization and deserialization via its [PolymorphicJsonAdapterFactory](https://github.com/square/moshi/blob/master/adapters/src/main/java/com/squareup/moshi/adapters/PolymorphicJsonAdapterFactory.java)

Before deserialization moshi need to be able to generate relevant adapters. This is done via `@JsonClass(generateAdapter = true)` annotation. For example
```kotlin
@JsonClass(generateAdapter = true)
data class Car(override val model: String,
               override val type: String,
               override val tyres: List<Tyre>?): Vehicle()
```

Next is setting up Moshi instance

```kotlin
val vehicleAdapterFactory = PolymorphicJsonAdapterFactory.of(Vehicle::class.java, "type")
        .withSubtype(Car::class.java, "car")
        .withSubtype(Truck::class.java, "truck")

val moshi = Moshi.Builder().add(vehicleAdapterFactory).build()
```

Now deserialize
```kotlin
try {
    val vehicleContainer = moshi.adapter(VehicleContainer::class.java)
        .fromJson(jsonStr, VehicleContainer.class)
} catch(error: JsonDataException) {
}
```

Here Moshi finds that second object in list does not respect language contract and hence, it fails the deserialization process. You will get a `JSONDataException` instead of expected `VehicleContainer` object.

This is a little nuance because in my case I still want to have objects which respect full contract like `Car` object in this example. We are using a modified version of this [SkipBadElementsListAdapter](https://stackoverflow.com/a/54190660/1107755) which solves the problem for us.

Moshi object now builds as follows
```
val moshi = Moshi.Builder()
    .add(vehicleAdapterFactory)
    .add(SkipBadElementsListAdapter.Factory)
    .build()
```

After this deserialization process we get a nice `vehicles` list inside `VehiclesContainer` object with one pristine object. 

Similarly you can use other adapters available [here](https://github.com/serj-lotutovici/moshi-lazy-adapters) and [here](https://github.com/square/moshi/tree/master/adapters) to customize moshi as per your needs or you can write your own.

In my experience while Moshi is some work to begin with first time. It really pays off in long term because of increased trust and quality in your own code. After a migration period of about 2 weeks we never had to deal with those unexpected `NullPointerException`'s because of somebody else's mistake. 

#### References/Further Readings
1. [Jake Wharton's Talk- A Few Ok Libraries](https://www.youtube.com/watch?v=WvyScM_S88c&feature=youtu.be)
2. [Moshi BenchMarks](https://github.com/ZacSweers/json-serialization-benchmarking)
3. [Gson Design Document](https://github.com/google/Gson/blob/master/GsonDesignDocument.md)
4. [Reddit Thread](https://www.reddit.com/r/androiddev/comments/684flw/why_use_moshi_over_Gson/)

Happy Coding!

Note: This article was *Originally published at [Why I ditched beloved Gson for my Kotlin project](https://medium.com/swlh/why-i-ditched-beloved-gson-for-my-kotlin-project-4acc1809fb68)*
