---
layout: post
title: "ReactFX's general stream combinator: State Machine"
tags: JavaFX ReactFX
---

In [ReactFX](http://www.reactfx.org), we work with event streams. It provides various operations on event streams such as `filter`, `map`, `merge`, `zip`, `combine`, `reduceSuccessions`. Of course these cannot cover all the use cases. In such a case, you can either extend [LazilyBoundStream](http://www.reactfx.org/javadoc/org/reactfx/LazilyBoundStream.html) to implement your custom stream combinator, or you can use the _state machine_ combinator.

A state machine has a _state_ and multiple _input streams_. An event emitted by an input stream causes the state machine to update its state and/or to emit an event. We demonstrate ReactFX's internal DSL (fluent API) for defining state machines on an example.

### Example

We implement a stream combinator that takes a source stream and two control streams. An event from one control stream _mutes_ the source stream, an event from the other control stream _unmutes_ the source stream. The resulting stream emits the same events as the source stream, but only when not muted. We are going to represent the internal state as a boolean indicating whether the source stream is muted.

{% highlight java linenos %}
EventStream<T> source = ...;
EventStream<?> muteImpulse = ...;
EventStream<?> unmuteImpulse = ...;

EventStream<T> combined = StateMachine.init(false)
    .on(muteImpulse).transition((wasMuted, i) -> true)
    .on(unmuteImpulse).transition((wasMuted, i) -> false)
    .on(source).emit((muted, t) -> muted ? Optional.empty() : Optional.of(t))
    .toEventStream();
{% endhighlight %}

* On line 5 we set the initial state to `false` (not muted).
* Line 6 says that when an event arrives from the `muteImpulse` stream, we set the internal state to `true` (muted), no matter what the previous state or the value of the impulse is.
* Line 7 says that when an event arrives from the `unmuteImpulse` stream, we set the internal state to `false` (not muted), no matter what the previous state or the value of the impulse is.
* Line 8 states that when an event arrives from the `source` stream, it is either emitted or not, based on the internal state (muted or not).
* Line 9 completes the creation of the state machine and the associated stream.
