---
layout: post
title: "Val&lt;T&gt;: a better ObservableValue"
tags: JavaFX ReactFX
---

[ObservableValue](http://docs.oracle.com/javase/8/javafx/api/javafx/beans/value/ObservableValue.html) is the JavaFX way of representing a time-varying value, whose changes can be observed by a listener. As it is, it really is a bare bones interface. I myself have previously enriched it in two different ways: EasyBind's `MonadicObservableValue` that adds some useful operations, and InhiBeans to postpone listener notifications. [Val](http://www.reactfx.org/javadoc/2.0-M2/org/reactfx/value/Val.html), introduced in ReactFX [2.0 Milestone 2](https://github.com/TomasMikula/ReactFX/releases/tag/v2.0-M2), integrates both these ideas and adds some more.


### Rich set of additional operations

Additional operations taken from EasyBind's [MonadicObservableValue]({% post_url 2014-03-26-monadic-operations-on-observablevalue %}), the most popular probably being `flatMap` a.k.a. [_type-safe select_](https://javafx-jira.kenai.com/browse/RT-35923):

```java
Val<Boolean> showing = Val.flatMap(node.sceneProperty(), Scene::windowProperty)
        .flatMap(Window::showingProperty);
```


### Suspendable listener notifications

InhiBeans' ability to [avoid glitches]({% post_url 2014-05-13-how-to-ensure-consistency-of-your-observable-state %}) by temporary suspension of listener notifications has been absorbed by `Val`:

```java
DoubleProperty width = new SimpleDoubleProperty(1.0);
DoubleProperty height = new SimpleDoubleProperty(2.0);
Val<Number> area = Val.suspendable(Bindings.multiply(width, height));
area.suspendWhile(() -> {
    width.set(3.0);
    height.set(4.0);
});
```

In this sample, any observer of `area` will only observe a single change from 2.0 to 12.0 (instead of a change from 2.0 to 6.0 and another change from 6.0 to 12.0).


### Lazily bound to dependencies

You can combine `Val`s to form new `Val`s. In JavaFX, this is known as creating _bindings_. There is one subtle but important difference in how these compound `Val`s bind to (i.e. start observing) their dependencies. Consider the following example:

```java
Val<Double> width = // ...
Val<Double> height = // ...
Val<Double> area = Val.combine(width, height, (w, h) -> w * h);
```

At this point, there is no listener registered with either `width` or `height`. When the first listener is attached to `area`, only at that point does `area` start observing `width` and `height`. Similarly, when the last listener is unregistered from `area`, `area` stops observing `width` and `height`. I call this _lazy binding_, for the lack of a better name. This should be familiar to ReactFX users, because this is how `EventStream`s have always done binding to their inputs. This concept is also known as _cold observables_ in Rx.

In the [previous post]({% post_url 2015-02-10-the-trouble-with-weak-listeners %}) I promised to give a better alternative to using `WeakListener`s for bindings. _Lazy binding_ is that alternative. No matter how big the network of bindings is, you only need to remove those listeners that you have registered. All intermediate links will be "disposed" automatically. No weak listeners needed, both problems mentioned in the previous post avoided.


### Correct behavior on recursive changes

It is possible that a change triggers another change of the same observable value. Changes reported by JavaFX's observable values in that case are simply _not correct_.

Let's show an example. The program below creates an `IntegerProperty` with initial value 0 and attaches two change listeners to it. The first change listener looks at the new value `val` and if it is greater than 0, (recursively) sets the property to `val - 1`. The second listener just prints the changes it observes to the standard output. Let's change the property value and see what happens.

```java
IntegerProperty p = new SimpleIntegerProperty(0);
p.addListener((obs, old, val) -> {
    if(val.intValue() > 0) {
        p.set(val.intValue() - 1);
    }
});
p.addListener((obs, old, val) -> System.out.println(old + " -> " + val));
p.set(2);
```

The output is

```
1 -> 0
2 -> 0
0 -> 0
```

which means that the second listener saw the value of the property changing from 1 to 0, then from 2 to 0, and finally from 0 to 0. **This is just wrong.** The value was actually changing like this: `0 -> 2 -> 1 -> 0`. Any of the following sequences of observed changes would be consistent with what happened:

<ul>
<li><pre>[0 -&gt; 2, 2 -&gt; 1, 1 -&gt; 0],</pre></li>
<li><pre>[0 -&gt; 2, 2     -&gt;     0],</pre></li>
<li><pre>[0     -&gt;     1, 1 -&gt; 0],</pre></li>
<li><pre>[                      ] <span style="color: gray"><i>(no change observed)</i></span>,</pre></li>
</ul>

but the listener observed something else.

Let's rewrite the above program using `Var`, writable variant of `Val`:

```java
Var<Integer> p = Var.newSimpleVar(0);
p.addListener((obs, old, val) -> {
    if(val > 0) {
        p.setValue(val - 1);
    }
});
p.addListener((obs, old, val) -> System.out.println(old + " -> " + val));
p.setValue(2);
```

The output of this program is

```
0 -> 1
1 -> 0
```

which is consistent with what actually happened.
