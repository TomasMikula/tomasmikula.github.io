---
layout: post
title: "Animated Transitions Made Easy"
tags: JavaFX ReactFX
---

In the [previous post]({% post_url 2015-02-10-val-a-better-observablevalue %}) I presented ReactFX's [Val](http://www.reactfx.org/javadoc/2.0-M3/org/reactfx/value/Val.html) as an improved [ObservableValue](http://docs.oracle.com/javase/8/javafx/api/javafx/beans/value/ObservableValue.html). In ReactFX 2.0 Milestone 3 it got even better: changes to values can be animated seamlessly.


This work was initially inspired by Mike Hearn's [animatedBind](https://gist.github.com/mikehearn/f639176566d735b676e7) function, but the API is rather different.

### TL;DR Version

In short, in order to make a value animate, you just need to specify the _duration_ of the transition and the _interpolation_ for the animated value.

```java
Val<Double> val = ...;
Val<Double> animated = val.animate(Duration.ofMillis(500), EASE_BOTH_DOUBLE);
```

Use adapter static methods if you don't yet have a `Val`:

```java
ObservableValue<Double> val = ...;
Val<Double> animated = Val.animate(val, Duration.ofMillis(500), EASE_BOTH_DOUBLE);
```

### Example: Animating Circle Center

We will be changing the center of a circle and we want the circle to move _smoothly_ to the new location, instead of jumping there right away. So we have a circle and a property that we will be changing in order to move the circle:

```java
Circle circle = new Circle(30.0, Color.BLUE);
Var<Point2D> center = Var.newSimpleVar(new Point2D(15.0, 15.0));
```

Remember, `Var` is just a `Property` with some additional methods. We are now going to define a value that transitions _smoothly_ to the value of `center` whenever `center` changes; call it `animCenter`:

{% highlight java linenos %}
Val<Point2D> animCenter = center.animate(
        Duration.ofMillis(400),
        (p1, p2, frac) -> p1.multiply(1.0-frac).add(p2.multiply(frac)));
{% endhighlight %}

* On line 1, `animate` is one of those additional methods defined on `Val`/`Var`.
* On line 2, we specified the duration of the transition.
* On line 3, we defined the interpolation between the start and end point. It is a linear interpolation between `p1` and `p2`.

Now we just need to bind the circle center to the animated center and we are all set. This needs to be done in two steps---binding the _x_ coordinate and binding the _y_ coordinate:

```java
circle.centerXProperty().bind(animCenter.map(Point2D::getX));
circle.centerYProperty().bind(animCenter.map(Point2D::getY));
```

Et voilà, the circle now transitions smoothly to the new position.

### Constant Duration vs. Constant Speed

One thing you may notice in the example above is that the circle moves faster when the new position is far from the current position than when it is close to the current position. This is because we specified constant duration of 400 milliseconds for each transition, no matter how far the circle has to travel. If we instead want a constant speed of the circle, we can define the duration as a function of the start and end points:

```java
Val<Point2D> animCenter = center.animate(
        (p1, p2) -> Duration.ofMillis((long) p1.distance(p2)),
        (p1, p2, frac) -> p1.multiply(1.0-frac).add(p2.multiply(frac)));
```

The only difference is on the second line, where now the duration of the transition is proportional to the distance between the two points. Now the speed of the circle will be the same for short and long distances, but it will take longer to travel longer distances.

Here is the full [source code of this demo](https://github.com/TomasMikula/ReactFX/blob/master/reactfx-demos/src/main/java/org/reactfx/demo/AnimatedValDemo.java), where clicking on the circle will relocate it to a random position.

### Does it animate even if no one is listening?

No! If there is no observer of the animated value, there is no animation going on in the background and no system resources are used. Just saying

```java
Val<T> animated = val.animate(duration, interpolator);
```

does not start the animation yet. Only when a listener is registered with the `animated` value does a JavaFX animation start in the background. As soon as all the listeners are removed, the background animation is stopped. In other words, animated values follow the ReactFX's general philosophy of _lazy binding_.
