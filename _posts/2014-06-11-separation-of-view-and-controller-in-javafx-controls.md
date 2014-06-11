---
layout: post
title: "Separation of View and Controller in JavaFX Controls"
tags: JavaFX
---

This post first really quickly introduces skins for JavaFX controls, then discusses the non-public base class for skin implementations (`BehaviorSkinBase`) and its pitfalls, and finally outlines how public API for Skin implementations that encourages separation of view and controller could look. This post presents a lesson learned from my recent refactoring of [RichTextFX](http://www.fxmisc.org/richtext/).

### Skins

I cannot do a better job of introducing skins than the Javadoc of the [javafx.scene.control](http://docs.oracle.com/javase/8/javafx/api/javafx/scene/control/package-summary.html#package.description) package:

> Controls follow the classic MVC design pattern. The [`Control`](http://docs.oracle.com/javase/8/javafx/api/javafx/scene/control/Control.html) is the "model".
> It contains both the state and the functions which manipulate that state.
> The Control class itself does not know how it is rendered or what the user
> interaction is. These tasks are delegated to the [`Skin`](http://docs.oracle.com/javase/8/javafx/api/javafx/scene/control/Skin.html) ("view"), which may
> internally separate out the view and controller functionality into separate
> classes, although at present there is no public API for the "controller"
> aspect.
>
> [...] It is also the responsibility of the Skin, or a delegate of the Skin,
> to implement and respond to all relevant key events which occur on the Control
> when it contains the focus.

Illustrated, it looks like this:

![Control-Skin relationship]({{ site.url }}/assets/img/2014-06-11-separation-of-view-and-controller-in-javafx-controls/public-api.png)

The skin _observes_ and _acts on_ the control, where, "observes" means listens to events (e.g. mouse events, key events), property changes, observable collections changes, and, more generally, calls functions that do not alter the control's state; "acts on" means calls functions to manipulate the control's state. The arrow also means that the skin holds a reference to the control, but not the other way around.<sup>[1)](#footnote-1)</sup>

The documentation further suggests to separate the view and controller functionality into separate classes. It does not currently provide any API for this, neither does it say anything about the relationship between the view and the controller:

![Control(Model)-View-Controller]({{ site.url }}/assets/img/2014-06-11-separation-of-view-and-controller-in-javafx-controls/recommendation.png)

### BehaviorSkinBase

Skins in JavaFX extend from `BehaviorSkinBase`, which is not part of the public API.<sup>[2)](#footnote-2)</sup> `BehaviorSkinBase` takes care of the view functionality, and delegates the controller functionality to _Behavior_. BehaviorSkinBase constructor takes a `Behavior` instance and makes it visible to subclasses, so that they can invoke methods on the behavior. Thus with BehaviorSkinBase, the undefined relationship from the previous diagram is "View acts on Controller" (which, in turn, acts on the model):

![BehaviorSkinBase]({{ site.url }}/assets/img/2014-06-11-separation-of-view-and-controller-in-javafx-controls/bsb.png)

From my limited exposure to skin implementations, the possibility of view invoking methods on the controller is rarely utilized, and where it is, the code could probably benefit from some refactoring (as did RichTextFX).

### The problem with BehaviorSkinBase

A real complication arises when the controller, in order to implement proper behavior of the control, needs to get some additional information from the view. Consider a text area. The model of a text area knows the caret position in terms of the index in the text, but does not know the caret coordinates on the screen or how paragraphs are broken into lines. To implement `Up`/`Down` key navigation, the controller needs to update the model with the new caret position (new index in the text). To this end, the controller needs to get additional information from the skin, because the new caret position depends on the current screen coordinates of the caret, as well as on how paragraphs are currently broken into lines.

![BehaviorSkinBase problem]({{ site.url }}/assets/img/2014-06-11-separation-of-view-and-controller-in-javafx-controls/bsb-problem.png)

This means there needs to be a back reference from the controller to the view. In the previous paragraph we have seen that this reference cannot, in general, be eliminated. On the other hand, I have come to believe that the other reference, the one from the view to the controller, can and should be eliminated.

### Alternative architecture

That leaves us with this architecture that we should be implementing:

![Ideal architecture]({{ site.url }}/assets/img/2014-06-11-separation-of-view-and-controller-in-javafx-controls/ideal.png)

The only difference from the BehaviorSkinBase architecture above is that we flipped one arrow.

Now let's put together some API that encourages this design. Let's have interface `Visual` for views and interface `Behavior` for controllers. Let's have a _final_ class `BehaviorSkin` that implements `Skin` and the only thing it does is that it creates the Visual and the Behavior.

```java
public interface Visual {
    Node getNode();
    void dispose();
    List<CssMetaData<? extends Styleable, ?>> getCssMetaData();
}

public interface Behavior {
    void dispose();
}

public final class BehaviorSkin<C extends Control, V extends Visual>
extends SkinBase<C> {
    private final V visual;
    private final Behavior behavior;
    private final Node node;

    public BehaviorSkin(
            C control,
            Function<? super C, ? extends V> visualFactory,
            BiFunction<? super C, ? super V, ? extends Behavior> behaviorFactory) {
        super(control);
        this.visual = visualFactory.apply(control);
        this.behavior = behaviorFactory.apply(control, visual);
        this.node = visual.getNode();
        getChildren().add(node);
    }

    @Override
    public void dispose() {
        getChildren().remove(node);
        behavior.dispose();
        visual.dispose();
    }


    @Override
    public List<CssMetaData<? extends Styleable, ?>> getCssMetaData() {
        return visual.getCssMetaData();
    }
}
```

The general template for the Control's [createDefaultSkin()](http://docs.oracle.com/javase/8/javafx/api/javafx/scene/control/Control.html#createDefaultSkin--) method will look like this:

```java
protected Skin<?> createDefaultSkin() {
    return new BehaviorSkin<>(
            this,
            control -> new FooVisual(control),
            (control, visual) -> new FooBehavior(control, visual));
}
```

or more concisely using constructor references

```java
protected Skin<?> createDefaultSkin() {
    return new BehaviorSkin<>(this, FooVisual::new, FooBehavior::new);
}
```

Note that the factory function for Visual gets only the control as the argument, while the factory function for Behavior gets the control as well as the Visual.

Also note that `BehaviorSkin` does not implement `Skin` directly, but instead extends [`SkinBase`](http://docs.oracle.com/javase/8/javafx/api/javafx/scene/control/SkinBase.html), which is already part of the public API.

The Behavior implementation may still extend from `BehaviorBase` where it did before, though this class is not part of the public API either.

### Conclusion

The outlined extension to the API for skin implementations is rather conservative---it does not present any radical shifts. Its main goal is to encourage separation of view and controller aspects. It does not address other challenges connected with skin implementations, such as a mechanism to bind key events to actions (which would be the responsibility of the BehaviorBase class). On the other hand, it does not close any doors to resolving such challenges.


----------
<a name="footnote-1">1)</a> Neglecting the fact that when attached to the scene, the control holds a reference to its skin, but as the author of the control, you don't ever access the skin from the control.

<a name="footnote-2">2)</a> Not being part of the public API means that you should not use `BehaviorSkinBase` in your applications. Anyway, it is used by JavaFX itself and some version of it may make it to the public API in the future, so I find it worth discussing.
