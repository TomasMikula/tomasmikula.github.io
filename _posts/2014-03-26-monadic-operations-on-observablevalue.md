---
layout: post
title: Monadic operations on ObservableValue
tags: JavaFX EasyBind
---

[EasyBind](http://www.fxmisc.org/easybind/) introduces [MonadicObservableValue](http://www.fxmisc.org/easybind/javadoc/org/fxmisc/easybind/monadic/MonadicObservableValue.html) that adds monadic operations to JavaFX [ObservableValue](http://docs.oracle.com/javase/8/javafx/api/javafx/beans/value/ObservableValue.html). First of all, don't get scared by the word _monadic_---it just means that there are a few more operations that treat `null` as "_value not available._" Anyway, if you have already used [Optional](http://docs.oracle.com/javase/8/docs/api/java/util/Optional.html), this will all look very familiar.

To get an initial instance of `MonadicObservableValue`, you would wrap a normal ObservableValue:

```java
ObservableValue<T> obs = ...;
MonadicObservableValue<T> monadic = EasyBind.monadic(obs);
```

Let's have a look at the additional methods available on MonadicObservableValue:

```java
interface MonadicObservableValue<T> extends ObservableValue<T> {
    boolean isPresent();
    boolean isEmpty();
    void ifPresent(Consumer<? super T> f);
    T getOrThrow();
    T getOrElse(T other);
    MonadicBinding<T> orElse(T other);
    MonadicBinding<T> orElse(ObservableValue<T> other);
    MonadicBinding<T> filter(Predicate<? super T> p);
    <U> MonadicBinding<U> map(Function<? super T, ? extends U> f);
    <U> MonadicBinding<U> flatMap(Function<? super T, ObservableValue<U>> f);
    <U> SelectBuilder<U> select(Function<? super T, ObservableValue<U>> f);
}
```

Note that all of these methods have _default_ implementation provided by the interface, so they would make a backwards compatible addition to `ObservableValue`.

**isPresent** checks whether this ObservableValue holds a _non-null_ value. 

**isEmpty** is the negation of `isPresent`. 

**ifPresent** executes an action if this ObservableValue holds a non-null value.

**getOrThrow** returns the current value of this ObservableValue. It throws an exception if the value is not available. 

**getOrElse** tries to return the current value of this ObservableValue. If a value is not available, returns the given fallback value. 

**orElse(`T`)** creates a new ObservableValue that holds the same value as this ObservableValue, but falls back to the provided default when this ObservableValue is empty. 

**orElse(`ObservableValue<T>`)** creates a new ObservableValue that holds the same value as this ObservableValue, but falls back to the value of the provided ObservableValue when this ObservableValue is empty. 

**filter** creates a new ObservableValue that holds the same value as this ObservableValue when the value satisfies the predicate and is empty when this ObservableValue is empty or its value does not satisfy the given predicate. 

**map** creates a new ObservableValue that holds a mapping of the value held by this ObservableValue, and is empty when this ObservableValue is empty. 

**flatMap(`f`)** creates a new ObservableValue that, when this ObservableValue holds value `x`, holds the value of ObservableValue `f(x)`, and is empty when this ObservableValue is empty.

**select** opens a selection chain, which is just a slightly more efficient equivalent to a chain of flatMaps. See the example below.


### Examples

Observable `Color` value that is either a user-picked color, or the background color of a certain region, or the default value:

```java
MonadicObservableValue<Color> userColor =
        EasyBind.monadic(colorPicker.valueProperty());

Binding<Color> bgColor = EasyBind.monadic(region.backgroundProperty())
        .filter(bg -> !bg.getFills().isEmpty())    // at least one background fill
        .map(bg -> bg.getFills().get(0).getFill()) // get the first background fill
        .filter(fill -> fill instanceof Color)     // background fill must be a color
        .map(fill -> (Color) fill);                // cast background fill to Color

Binding<Color> color = userColor.orElse(bgColor).orElse(AQUA);
```

Current window of a node:

```java
Binding<Window> window = EasyBind.monadic(node.sceneProperty())
        .flatMap(Scene::windowProperty);
```

Indicator of whether a node is being shown:

```java
Binding<Boolean> showing = EasyBind.monadic(node.sceneProperty())
        .flatMap(Scene::windowProperty)
        .flatMap(Window::showingProperty);
```

When there is a chain of flatMaps as above, it is slightly more efficient to use a selection chain instead:

```java
Binding<Boolean> showing = EasyBind.monadic(node.sceneProperty())
        .select(Scene::windowProperty)
        .selectObject(Window::showingProperty);
```

The only difference is that in a chain of flatMaps, every `flatMap` creates a Binding instance, while in a selection chain, only the terminal `selectObject` creates a Binding instance.
