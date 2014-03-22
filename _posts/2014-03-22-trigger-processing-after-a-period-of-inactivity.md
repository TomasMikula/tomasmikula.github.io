---
layout: post
title: Trigger processing after a period of inactivity
tags: JavaFX ReactFX
---

This post shows how to use [ReactFX](http://www.reactfx.org) to defer processing of user input until a specified period of user's inactivity. This is useful, for example, to trigger spell checking or syntax highlighting after the user hasn't typed anything for, say, 500ms. Another use-case, which we use in this post, is real-time HTML rendering of user input.

Imagine an application with a TextArea for raw HTML input and a WebView where this HTML is rendered in real time.

```java
TextArea textArea = new TextArea();
WebView webView = new WebView();
WebEngine engine = webView.getEngine();
```

To set up real-time rendering of raw HTML input, you would normally do this:

```java
textArea.textProperty().addListener(
        (obs, oldHtml, newHtml) -> engine.loadContent(newHtml));
```

This works fine, but it causes the HTML to be re-rendered on every keystroke.

### Task

Let's render the HTML only after the input hasn't changed for 500 milliseconds. This will relieve your CPU quite a bit.

### Solution

This is how to achieve this with ReactFX:

{% highlight java linenos %}
EventStreams.valuesOf(textArea.textProperty())
        .reduceCloseSuccessions((a, b) -> b, Duration.ofMillis(500))
        .subscribe(html -> engine.loadContent(html));
{% endhighlight %}

### Explanation

 1. The first line creates an _event stream_ that emits the new value of input every time the input text changes.

 2. The second line creates an event stream that _reduces_ values from the first stream that come in close temporal succession into one. The reduction function `(a, b) -> b` means that we always retain the latter of the two values. After arrival of a value from the first stream, this stream waits for at most 500 milliseconds for another value. If a value arrives within this interval, the two values are reduced (here meaning that the previous one is forgotten) and the stream waits for another 500ms. If no value arrives within this interval, the currently stored value is emitted from this stream.

 3. The third line says that whenever a value (raw HTML) is emitted from the second stream, we pass it to the web engine for rendering.

### Full example

```java
import java.time.Duration;

import javafx.application.Application;
import javafx.scene.Scene;
import javafx.scene.control.SplitPane;
import javafx.scene.control.TextArea;
import javafx.scene.web.WebEngine;
import javafx.scene.web.WebView;
import javafx.stage.Stage;

import org.reactfx.EventStreams;

public class DeferredHtmlRendering extends Application {

    public static void main(String[] args) {
        launch(args);
    }

    @Override
    public void start(Stage primaryStage) {
        TextArea textArea = new TextArea();
        WebView webView = new WebView();
        WebEngine engine = webView.getEngine();

        EventStreams.valuesOf(textArea.textProperty())
                .reduceCloseSuccessions((a, b) -> b, Duration.ofMillis(500))
                .subscribe(html -> engine.loadContent(html));

        SplitPane root = new SplitPane();
        root.getItems().addAll(textArea, webView);
        Scene scene = new Scene(root);
        primaryStage.setScene(scene);
        primaryStage.show();
    }
}
```
