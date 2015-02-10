---
layout: post
title: "The Trouble with Weak Listeners"
tags: JavaFX
---

JavaFX bindings use [WeakListener](http://docs.oracle.com/javase/8/javafx/api/javafx/beans/WeakListener.html)s to observe their dependencies. This reduces the need to [dispose()](http://download.java.net/jdk8/jfxdocs/javafx/beans/binding/Binding.html#dispose--) bindings, but comes with some surprising behavior. In this post I show two kinds of problems introduced by weak listeners. In the next post, I will show how we can do better without weak listeners.


### Why Weak Listeners

(This section is my attempt to "reverse-engineer" the motivation behind weak listeners. This will be familiar to most readers, so feel free to skip to the next section.)

In a straightforward implementation, a [Binding](http://download.java.net/jdk8/jfxdocs/javafx/beans/binding/Binding.html) would need to be `dispose()`d in order to stop observing its dependencies and become eligible for garbage collection. This could get really cumbersome when you have something like this:

```java
DoubleExpression x = // ...
DoubleBinding result = Bindings.max(x.add(5.0).multiply(7.0), 100.0);
```

In order to dispose the above binding correctly, it is not sufficient to do `result.dispose()`---one would need to dispose all the intermediate bindings as well:

```java
DoubleExpression x = // ...
DoubleBinding intermediate1 = x.add(5.0);
DoubleBinding intermediate2 = intermediate.multiply(7.0);
DoubleBinding result = Bindings.max(intermediate2, 100.0);

// dispose
intermediate1.dispose();
intermediate2.dispose();
result.dispose();
```

(I know, I know, it should be sufficient to dispose `intermediate1`, but still...)

As you can see, this is very impractical, so there is a trick: weak listeners! In the above example, `x` holds only a [weak reference](http://docs.oracle.com/javase/8/docs/api/java/lang/ref/WeakReference.html) to `intermediate1`, which in turn holds a weak reference to `intermediate2`, which holds a weak reference to `result`. Hence, `dispose()` is optional---bindings will be garbage collected when they are no longer strongly reachable.


### Problem 1: Accidental Garbage Collection

```java
DoubleExpression x = new SimpleDoubleProperty(0.0);
Bindings.max(x.add(5.0).multiply(7.0), 100.0).addListener(listener);
x.set(1.0);
```

Guess what: the `listener` may never be executed. Surprise! All the intermediate bindings may be garbage collected right away. For most readers, this is old news. And they already know that they need to store a reference to the resulting binding to prevent garbage collection:

```java
DoubleExpression x = new SimpleDoubleProperty(0.0);
this.result = Bindings.max(x.add(5.0).multiply(7.0), 100.0);
this.result.addListener(listener);
x.set(1.0);
```

Still, this keeps biting people over and over.


### Problem 2: Memory Leaks

This problem is more subtle and less known. That's right, weak listeners do not entirely prevent memory leaks. A weak listener is a wrapper that holds a weak reference to the actual (non-weak) listener. It is true that when the actual listener is no longer strongly reachable, it will be garbage collected. But what about removing the wrapper (holding the now cleared weak reference) from the list of listeners of an observable? The wrapper clears itself when it is invoked and finds out its actual listener is gone. The problem arises when the observable is never invalidated again, and thus the weak listener does not get a chance to clear itself. Here is an example that amplifies the problem by registering many weak listeners to an observable that is never invalidated:

{% gist TomasMikula/62c6e33863f2092f27c9 Leaky.java %}

The output of this program is

```
Used Memory after GC: 52MB
```
