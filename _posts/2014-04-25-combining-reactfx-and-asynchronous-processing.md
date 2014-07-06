---
layout: post
title: Combining ReactFX and asynchronous processing
tags: JavaFX ReactFX
---

We have seen previously how to use [ReactFX](http://www.reactfx.org) to [trigger processing after a period of user's inactivity]({% post_url 2014-03-22-trigger-processing-after-a-period-of-inactivity %}). In this post, we are going to make the processing asynchronous and apply its result back to the scene when complete. The example we are going to use is _syntax highlighting_.

### Task

When the user pauses typing for 500 milliseconds, trigger background computation of syntax highlighting. When done, apply the computed highlighting to the text, but only if it's still valid, i.e. the text hasn't changed in the meantime.

### Prerequisites

In order to make syntax highlighting work, we need a text area that supports highlighting ranges of text, such as [RichTextFX](http://www.fxmisc.org/richtext/).

We are going to use JavaFX's [Task](http://docs.oracle.com/javase/8/javafx/api/javafx/concurrent/Task.html) to represent asynchronous computations. (In addition to Task, ReactFX also supports [CompletionStage](http://docs.oracle.com/javase/8/docs/api/java/util/concurrent/CompletionStage.html) as a representation of asynchronous computation.)

### Assumptions

We don't include the details of how to represent, compute and apply highlighting. We are just going to assume the following definitions:

```java
/** Represents highlighting. */
class Highlighting {
    // ...
}

/** Computes highlighting for the given text. */
Highlighting computeHighlighting(String text) {
    // ...
}

/** Applies the given highlighting to the text area. */
void applyHighlighting(Highlighting highlighting) {
    // ...
}
```

### Helpers

We define a helper method that computes highlighting in the background.

```java
ExecutorService executor = Executors.newSingleThreadExecutor();

Task<Highlighting> computeHighlightingAsync(String text) {
    Task<Highlighting> task = new Task<Highlighting>() {
        @Override
        protected Highlighting call() {
            return computeHighlighting(text);
        }
    };
    executor.execute(task);
    return task;
}
```

### Solution

{% highlight java linenos %}
EventStream<String> textValues = EventStreams.valuesOf(textArea.textProperty());
EventStream<?> cancelImpulse = textValues;
textValues.successionEnds(Duration.ofMillis(500))
        .mapToTask(this::computeHighlightingAsync)
        .awaitLatest(cancelImpulse)
        .subscribe(this::applyHighlighting);
{% endhighlight %}

### Explanation

 1. Line 1 creates an _event stream_ that emits the new value of text area's text every time the text changes.

 2. Line 2 just gives another name to the same stream. We are going to use every new value of text to cancel the asynchronous computation currently in progress.

 3. On line 3, we only keep the text values that haven't changed in the last 500 milliseconds.

 4. Line 4 starts asynchronous computation of highlighting for text values that survived the filter from line 3.

 5. Line 5 creates a stream that emits the results of asynchronous computations started by line 4. It only emits the ones that haven't been outdated by the time they are completed. An asynchronous computation is considered outdated if either another asynchronous computation starts before it is completed, or an impulse to cancel the computation arrives. As a consequence, a highlighting that is emitted from the stream on line 5 is up to date and can be applied to the text in text area.

 6. Line 6 just applies every highlighting emitted on line 5 to the text in the text area.

**EDIT:** Here is a marble diagram from [Eugen Kiss](https://github.com/eugenkiss) illustrating the possible situations.
![Marble diagram]({{ site.url }}/assets/img/2014-04-25-combining-reactfx-and-asynchronous-processing/marble-diagram.png)

### Full example

A full working example is [this demo](https://github.com/TomasMikula/RichTextFX/blob/master/richtextfx-demos/src/main/java/org/fxmisc/richtext/demo/JavaKeywordsAsync.java) that highlights Java keywords.
