---
layout: post
title: "How to ensure consistency of your observable state"
tags: JavaFX ReactFX
---

In this post, I would like to make an appeal to JavaFX developers, especially library creators, to provide stronger consistency guaranties of their observable objects.


### The problem

When implementing an object with two or more observable properties that should satisfy a certain invariant (i.e. they are not completely independent), it is often tricky to ensure that the invariant holds all the time the client code is able to query the values of the properties. The invariant often breaks for a brief moment when updating property values: a listener of the property to be updated first can observe the old state of the property to be updated next. As a consequence, the client code needs to be more defensive (i.e. _longer_ and _less clear_) to handle potential inconsistencies.

We demonstrate the issue on [TextArea](http://docs.oracle.com/javase/8/javafx/api/javafx/scene/control/TextArea.html). It has (among others) these properties: `caretPosition`, `anchor`, `selection`. There is a natural relationship that should hold between these three properties:

 * _selection = (anchor, caretPosition),_ **if** _anchor &le; caretPosition_;
 * _selection = (caretPosition, anchor),_ **otherwise.**

Stated in code, it is

```java
selection.equals(IndexRange.normalize(anchor, caretPosition))
```

So let's write a program that tests the above invariant:

```java
TextArea area = new TextArea();
area.caretPositionProperty().addListener(obs -> checkConsistency(area));
area.anchorProperty().addListener(obs -> checkConsistency(area));
area.selectionProperty().addListener(obs -> checkConsistency(area));

void checkConsistency(TextArea area) {
    IndexRange selection = area.getSelection();
    int caretPosition = area.getCaretPosition();
    int anchor = area.getAnchor();
    assert selection.equals(IndexRange.normalize(anchor, caretPosition));
}
```

Try the [full runnable version](https://gist.github.com/TomasMikula/70f085c36c21ce964f07). Don't forget to enable assertions. Run the program and type a letter to the text area. You will get an AssertionError, which means the tested invariant has been broken.


### Can we fix it?

Yes!

[InhiBeans](https://github.com/TomasMikula/ReactFX/wiki/InhiBeans), a subpackage of [ReactFX](http://www.reactfx.org), provides property and binding implementations that allow you to update their value or invalidate them without notifying the listeners right away. You can update a number of properties and only then notify the listeners. By the time a listener of property _p_ queries the value of property _q_, both properties will have been updated and thus consistent.

Here is a sketch of how one could fix the above inconsistency in TextArea:

1. Switch the implementation of `caretPosition`, `anchor` and `selection` properties for their counterpart from `org.reactfx.inhibeans`.
2. Replace any `selectionChangingOperation()` on TextArea (such as typing, moving the caret, ...) by

    ```java
    try(
        Guard g1 = caretPosition.block();
        Guard g2 = anchor.block();
        Guard g3 = selection.block()
    ) {
        selectionChangingOperation();
    }
    ```

    or, more concisely

    ```java
    Guardian.combine(caretPosition, anchor, selection)
            .guardWhile(() -> selectionChangingOperation());
    ```

**Note:** In some cases, you can provide consistency guaranties just by arranging the order of invalidation propagation among dependent properties and/or bindings. This solution, however, relies on the implementation detail that property and binding implementations provided by JavaFX itself notify the listeners in the order they were registered. It is therefore fragile to future changes or alternative property/binding implementations. Also, it gets complicated very quickly.


### What about observable collections?

InhiBeans currently provide a [wrapper for `ObservableList`](http://www.reactfx.org/javadoc/org/reactfx/inhibeans/collection/Collections.html#wrap-javafx.collections.ObservableList-) that is able to temporarily block the notifications of list changes. All changes made to the list while blocked are combined into a single list change that is fired upon release. Blockable wrappers for other collections might be added in the future.
