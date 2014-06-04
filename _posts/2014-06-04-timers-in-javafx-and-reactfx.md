---
layout: post
title: "Timers in JavaFX and ReactFX"
tags: JavaFX ReactFX
---

The takeaway from this post should be that to schedule future actions in a JavaFX application _you don't need an external timer_ (and thus an extra thread in your application), such as `java.util.Timer` or `java.util.concurrent.ScheduledExecutorService`. You can use [Timeline](http://docs.oracle.com/javase/8/javafx/api/javafx/animation/Timeline.html) or [FxTimer](http://www.reactfx.org/javadoc/org/reactfx/util/FxTimer.html), a convenient wrapper around Timeline used internally in [ReactFX](http://www.reactfx.org/) and recently exposed as public API.


### One-time action

This is how to use `Timeline` to schedule a one-time action:

```java
Timeline timeline = new Timeline(new KeyFrame(
        Duration.millis(2500),
        ae -> doSomething()));
timeline.play();
```

Equivalently, you can use this ReactFX shortcut:

```java
FxTimer.runLater(
        Duration.ofMillis(2500),
        () -> doSomething());
```

In addition to the latter being more readable, `FxTimer` uses the new [Java Time API](http://docs.oracle.com/javase/8/docs/api/java/time/package-summary.html) introduced in JDK 8.


### Periodic action

This is how to schedule a periodic action using `Timeline`:

```java
Timeline timeline = new Timeline(new KeyFrame(
        Duration.millis(2500),
        ae -> doSomething()));
timeline.setCycleCount(Animation.INDEFINITE);
timeline.play();
```

Using `FxTimer`, it looks like this:

```java
FxTimer.runPeriodically(
        Duration.ofMillis(2500),
        () -> doSomething());
```

Or, because we really like event streams:

```java
EventStreams.ticks(Duration.ofMillis(2500))
        .subscribe(tick -> doSomething());
```


### Timer cancellation

Let's see how one can cancel a scheduled action.

The `Timeline` way:

```java
Timeline timeline = new Timeline(new KeyFrame(
        Duration.millis(2500),
        ae -> doSomething()));
timeline.play();

// later
timeline.stop();
```

The `FxTimer` way:

```java
Timer timer = FxTimer.runLater(
        Duration.ofMillis(2500),
        () -> doSomething());

// later
timer.stop();
```

There is one _important semantic difference_ between the above code snippets. In the first one, `doSomething()` may still be executed _after_ `timeline.stop()`, even though both `timeline.stop()` and `doSomething()` are executed on the JavaFX application thread. It happens when `stop()` is called after the timeline has reached the key frame, but before the associated action had a chance to be executed on the JavaFX application thread. **EDIT:** Note that this behavior makes sense for animations: if there is an action _a_ associated with time _t1_ and `stop()` is called at time _t2 > t1_, then, if not already done so, _a_ should still be executed in order to advance the animation to a state corresponding to _t2_.

On the other hand, for timers it makes sense to cancel the timer upon `stop()` even if the timer is already overdue. Therefore, `FxTimer` takes an extra measure to ensure that the action is never executed after `stop()`.

<details>
  <summary>Here is the code that shows that behavior.</summary>
  {% gist TomasMikula/f086616d0aa617de6e74 %}
</details>
