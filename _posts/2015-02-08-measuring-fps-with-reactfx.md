---
layout: post
title: "Measuring FPS with ReactFX"
tags: JavaFX ReactFX
---

I came across this [StackOverflow question about measuring frame rate in JavaFX](http://stackoverflow.com/questions/28287398/what-is-the-preferred-way-of-getting-the-frame-rate-of-a-javafx-application) and while James_D gave a nice solution, I still found it a little too verbose for such a simple task.

So I just added one more feature before releasing [ReactFX 2.0 Milestone 2](https://github.com/TomasMikula/ReactFX/releases/tag/v2.0-M2), namely [animationTicks()](http://www.reactfx.org/javadoc/2.0-M2/org/reactfx/EventStreams.html#animationTicks--). Using that, the solution now looks like this:

```java
EventStreams.animationTicks()
        .latestN(100)
        .map(ticks -> {
            int n = ticks.size() - 1;
            return n * 1_000_000_000.0 / (ticks.get(n) - ticks.get(0));
        })
        .map(d -> String.format("FPS: %.3f", d))
        .feedTo(label.textProperty());
```

Here is the full runnable [demo](https://github.com/TomasMikula/ReactFX/blob/master/reactfx-demos/src/main/java/org/reactfx/demo/FPSDemo.java), adapted from James' answer.
