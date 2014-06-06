---
layout: post
title: "Detecting when the mouse stays still over a node"
tags: JavaFX ReactFX
---

Everyone has seen tooltips: the mouse enters a node, stays still for a while, and then a tooltip is displayed. JavaFX does have [tooltips](http://docs.oracle.com/javase/8/javafx/api/javafx/scene/control/Tooltip.html). They are easy to use, but not very flexible. For example, you cannot control the delay before the tooltip is shown or for how long it is shown. If you want to roll your own tooltip implementation, the first step is to detect _when_ and _where_ the mouse stays still.

For touch devices, JavaFX has the [`TOUCH_STATIONARY`](http://docs.oracle.com/javase/8/javafx/api/javafx/scene/input/TouchEvent.html#TOUCH_STATIONARY) event. There's no such event for mouse, so we are going to simulate it by means of available mouse events.

#### Plan

Detect all mouse events on a node. When a `MOUSE_MOVED` event hasn't been followed by any mouse event for 1 second, we conclude that the mouse is stationary. When the next mouse event arrives, we conclude that the mouse stopped being stationary. We are going to create a [ReactFX](http://www.reactfx.org/) event stream that emits the mouse position when the mouse becomes stationary and `null` when it stops being stationary.

#### Solution

{% highlight java linenos %}
EventStream<MouseEvent> mouseEvents = eventsOf(node, MouseEvent.ANY);

EventStream<Point2D> stationaryPositions = mouseEvents
        .successionEnds(Duration.ofSeconds(1))
        .filter(e -> e.getEventType() == MouseEvent.MOUSE_MOVED)
        .map(e -> new Point2D(e.getX(), e.getY()));

EventStream<Void> stoppers = mouseEvents.supply(null);

EitherEventStream<Point2D, Void> stationaryEvents =
        stationaryPositions.or(stoppers)
                .distinct();
{% endhighlight %}

* Line 1 creates a stream of all mouse events on `node`.
* Line 4 keeps only events that haven't been followed by another one for 1 second.
* Line 5 further filters the stream to contain only mouse moves.  
<small>This means that, for example, if the last event is a click and then the mouse stays still, it is not detected as mouse being stationary. Note that this is consistent with how tooltips work---the tooltip is not displayed after the node is clicked.</small>
* Line 6 converts mouse events to mouse positions.
* Line 8 declares that any mouse event is a reason to end the stationary state.
* Line 11 includes both types of events (<q>started being stationary</q> and <q>stopped being stationary</q>) in one stream.
* Line 12 filters out repeating <q>stopped being stationary</q> events.

#### Highlights

* There are _no mutable variables_ to track state, like `lastMousePosition` or `isStationary`. All the variables above are effectively final and all state is managed by the event streams.
* There is _no CPU overhead_ when no one is actually subscribed to `stationaryEvents`. This is due to the lazy nature of event streams. Only when someone subscribes to `stationaryEvents` does this propagate all the way down and an event handler is registered for mouse events on the node.

#### Usage

```java
stationaryEvents.left().subscribe(pos -> showTooltipAt(pos));
stationaryEvents.right().subscribe(stop -> hideTooltip());
```


### Optionally: Dispatch `MouseStationaryEvent`s on a node

You may want to use JavaFX's standard [`addEventHandler`](http://docs.oracle.com/javase/8/javafx/api/javafx/scene/Node.html#addEventHandler-javafx.event.EventType-javafx.event.EventHandler-) method to detect when the mouse starts and stops being stationary.

Let's define `MouseStationaryEvent` with the following API:

```java
public class MouseStationaryEvent extends InputEvent {

    public static final EventType<MouseStationaryEvent> ANY;
    public static final EventType<MouseStationaryEvent> MOUSE_STATIONARY_BEGIN;
    public static final EventType<MouseStationaryEvent> MOUSE_STATIONARY_END;

    /** Creates a new event of type MOUSE_STATIONARY_BEGIN. */
    static final MouseStationaryEvent beginAt(Point2D screenPos);

    /** Creates a new event of type MOUSE_STATIONARY_END. */
    static final MouseStationaryEvent end();

    public Point2D getPosition();
    public Point2D getScenePosition();
    public Point2D getScreenPosition();
}
```

<details>
  <summary>Implementation of this class is not very interesting, but is provided here for completeness.</summary>
  {% gist TomasMikula/54bf9ce95ce6bc2f3a56 MouseStationaryEvent.java %}
</details>

Now, to start dispatching `MouseStationaryEvent`s for a node, you simply do this:

```java
EitherEventStream<Point2D, Void> stationaryEvents = // defined above

stationaryEvents.unify(
        pos -> MouseStationaryEvent.beginAt(node.localToScreen(pos)),
        stop -> MouseStationaryEvent.end())
    .subscribe(evt -> Event.fireEvent(node, evt));
```

The `unify` operator converts a stream of <q>_either `Point2D` or `Void`_</q> to a stream of a single type `MouseStationaryEvent`. `Point2D`s are converted to a `MouseStaionaryEvent` of type `MOUSE_STATIONARY_BEGIN` and `Void`s are converted to a `MouseStationaryEvent` of type `MOUSE_STATIONARY_END`.

#### Usage

```java
node.addEventHandler(MOUSE_STATIONARY_BEGIN, e -> {
    showTooltipAt(e.getScreenPosition());
});

node.addEventHandler(MOUSE_STATIONARY_END, e -> {
    hideTooltip();
});
```


### Convenient helper class

<details>
  <summary>Finally, here is a helper class combining all of the above that you can use in your projects.</summary>
  {% gist TomasMikula/54bf9ce95ce6bc2f3a56 MouseStationaryHelper.java %}
</details>

#### Usage

```java
// detect stationary events on a node after 1 second delay
MouseStationaryHelper helper =
        new MouseStationaryHelper(node, Duration.ofSeconds(1));

helper.events().left().subscribe(pos -> showTooltipAt(pos));
helper.events().right().subscribe(stop -> hideTooltip());

// If you would rather use standard JavaFX way,
// start dispatching MouseStationaryEvents on the node.
helper.install();

node.addEventHandler(MOUSE_STATIONARY_BEGIN, e -> {
    showTooltipAt(e.getScreenPosition());
});

node.addEventHandler(MOUSE_STATIONARY_END, e -> {
    hideTooltip();
});

// optionally, stop dispatching MouseStationaryEvents
helper.uninstall();
```
