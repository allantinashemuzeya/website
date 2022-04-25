---
title: Performance best practices
short-title: Best practices
description: How to ensure that your Flutter app is performant.
---

{% include docs/performance.md %}

Generally, Flutter applications are performant by default,
so you only need to avoid common pitfalls to get excellent
performance. These best practice recommendations will help you
write the most performant Flutter app possible.

{{site.alert.note}}
  If you are writing web apps in Flutter, you might be interested
  in a series of articles, written by the Flutter Material team,
  after they modified the [Flutter Gallery][] app to make it more
  performant on the web:

  * [Optimizing performance in Flutter web apps with tree
    shaking and deferred loading][web-perf-1]
  * [Improving perceived performance with image placeholders,
    precaching, and disabled navigation transitions][web-perf-2]
  * [Building performant Flutter widgets][web-perf-3]
{{site.alert.end}}

[Flutter Gallery]: {{site.gallery}}
[web-perf-1]: {{site.flutter-medium}}/optimizing-performance-in-flutter-web-apps-with-tree-shaking-and-deferred-loading-535fbe3cd674
[web-perf-2]: {{site.flutter-medium}}/improving-perceived-performance-with-image-placeholders-precaching-and-disabled-navigation-6b3601087a2b
[web-perf-3]: {{site.flutter-medium}}/building-performant-flutter-widgets-3b2558aa08fa

How do you design a Flutter app to most efficiently
render your scenes? In particular, how do you ensure
that the painting code generated by the
framework is as efficient as possible?
Some rendering and layout operations are known
to be slow, but can't always be avoided.
They should be used thoughtfully,
following the guidance below.

# Minimize expensive operations

Some operations are more expensive than others,
meaning that they consume more resources.
Obviously, you want to only use these operations
when necessary. How you design and implement your
app's UI can have a big impact on how efficiently it runs.

## Control build() cost

Here are some things to keep in mind when designing your UI:

* Avoid repetitive and costly work in `build()` methods
  since `build()` can be invoked frequently when
  ancestor widgets rebuild.
* Avoid overly large single widgets with a large `build()` function.
  Split them into different widgets based on encapsulation
  but also on how they change:
  * When `setState()` is called on a `State` object,
    all descendent widgets rebuild. Therefore,
    localize the `setState()` call to the part of
    the subtree whose UI actually needs to change.
    Avoid calling `setState()` high up in the tree
    if the change is contained to a small part of the tree.
  * The traversal to rebuild all descendents stops when the
    same instance of the child widget as the previous
    frame is re-encountered. This technique is heavily
    used inside the framework for optimizing
    animations where the animation doesn't affect the child subtree.
    See the [`TransitionBuilder`][] pattern and
    the [source code for `SlideTransition`][],
    which uses this principle to avoid rebuilding its
    descendents when animating.
  * Use `const` constructors on widgets as much as possible,
    since they allow Flutter to short-circuit most
    of the rebuild work. To be automatically reminded
    to use `const` when possible, enable the
    recommended lints from the [`flutter_lints`][] package.
    For more information, check out the
    [`flutter_lints` migration guide][].
  * To create reusable pieces of UIs,
    prefer using a [`StatelessWidget`][]
    rather than a function.

For more information, check out:

* [Performance considerations][],
  part of the [`StatefulWidget`][] API doc
* [Widgets vs helper methods][],
  a video from the official Flutter YouTube 
  channel that explains why widgets
  (especially widgets with `const` constructors)
  are more performant than functions.

[`flutter_lints`]: {{site.pub-pkg}}/packages/flutter_lints
[`flutter_lints` migration guide]: {{site.url}}/release/breaking-changes/flutter-lints-package#migration-guide
[Performance considerations]: {{site.api}}/flutter/widgets/StatefulWidget-class.html#performance-considerations
[source code for `SlideTransition`]: {{site.repo.flutter}}/blob/master/packages/flutter/lib/src/widgets/transitions.dart#L168
[`StatefulWidget`]: {{site.api}}/flutter/widgets/StatefulWidget-class.html
[`StatelessWidget`]: {{site.api}}/flutter/widgets/StatelessWidget-class.html
[`TransitionBuilder`]: {{site.api}}/flutter/widgets/TransitionBuilder.html
[Widgets vs helper methods]: {{site.youtube-site}}/watch?v=IOyq-eTRhvo

---

## Use saveLayer() thoughtfully

Some Flutter code uses `saveLayer()`, an expensive operation,
to implement various visual effects in the UI.
Even if your code doesn't explicitly call `saveLayer()`,
other widgets or packages that you use might call it behind the scenes.
Perhaps your app is calling `saveLayer()` more than necessary;
excessive calls to `saveLayer()` can cause jank.

### Why is saveLayer expensive?

Calling `saveLayer()` allocates an offscreen buffer
and drawing content into the offscreen buffer might
trigger a render target switch.
The GPU wants to run like a firehose,
and a render target switch forces the GPU
to redirect that stream temporarily and then
direct it back again. On mobile GPUs this is
particularly disruptive to rendering throughput.

### When is saveLayer required?

At runtime, if you need to dynamically display various shapes
coming from a server (for example), each with some transparency,
that might (or might not) overlap,
then you pretty much have to use `saveLayer()`.

### Debugging calls to saveLayer

How can you tell how often your app calls `saveLayer()`,
either directly or indirectly?
The `saveLayer()` method triggers
an event on the [DevTools timeline][]; learn when
your scene uses `saveLayer` by checking the
`PerformanceOverlayLayer.checkerboardOffscreenLayers`
switch in the [DevTools Performance view][].

[DevTools timeline]: {{site.url}}/development/tools/devtools/performance#timeline-events-chart

### Minimizing calls to saveLayer

Can you avoid calls to `saveLayer`?
It might require rethinking of how you
create your visual effects:

* If the calls are coming from _your_ code, can you
  reduce or eliminate them?
  For example, perhaps your UI overlaps two shapes,
  each having non-zero transparency:
  * If they always overlap in the same amount,
    in the same way, with the same transparency,
    you can precalculate what this overlapped,
    semi-transparent object looks like, cache it,
    and use that instead of calling `saveLayer()`. 
    This works with any static shape you can precalculate.
  * Can you refactor your painting logic to avoid
    overlaps altogether?
{% comment %}
TBD: It would be nice if we could link to an example.
  Kenzie suggested to John and Tao that we add an
  example to perf_diagnosis_demo. Michael indicated
  that he doesn't have a saveLayer demo.
{% endcomment %}

* If the calls are coming from a package that you don't own,
  contact the package owner and ask why
  these calls are necessary? Can they be reduced or
  eliminated? If not, you might need to find another
  package, or write your own.

{{site.alert.secondary}}
  **Note to package owners:**
  As a best practice, consider providing documentation
  for when `saveLayer` might be necessary for your package,
  how it might be avoided, and when it can't be avoided.
{{site.alert.end}}

Other widgets that might trigger `saveLayer()`
and are potentially costly:

* [`ShaderMask`][]
* [`ColorFilter`][]
* [`Chip`][]&mdash;might trigger a call to `saveLayer()` if
  `disabledColorAlpha != 0xff`
* [`Text`][]&mdash;might trigger a call to `saveLayer()`
  if there's an `overflowShader`

[`Chip`]: {{site.api}}/flutter/material/Chip-class.html
[`ColorFilter`]: {{site.api}}/flutter/dart-ui/ColorFilter-class.html
[`FadeInImage`]: {{site.api}}/flutter/widgets/FadeInImage-class.html
[`Opacity`]: {{site.api}}/flutter/widgets/Opacity-class.html
[`ShaderMask`]: {{site.api}}/flutter/widgets/ShaderMask-class.html
[`Text`]: {{site.api}}/flutter/widgets/Text-class.html
[Transparent image]: {{site.api}}/flutter/widgets/Opacity-class.html#transparent-image

---

## Minimize use of opacity and clipping

Opacity is another expensive operation, as is clipping.
Here are some tips you might find to be useful:

* Use the [`Opacity`][] widget only when necessary.
  See the [Transparent image][] section in the `Opacity`
  API page for an example of applying opacity directly
  to an image, which is faster than using the `Opacity`
  widget.
* Instead of wrapping simple shapes or text
  in an `Opacity` widget, it's usually faster to
  just draw them with a semitransparent color.
  (Though this only works if there are no overlapping
  bits in the to-be-drawn shape.)
* To implement fading in an image, consider using the
  [`FadeInImage`][] widget, which applies a gradual
  opacity using the GPU’s fragment shader.
  For more information, check out the [`Opacity`][] docs.
* **Clipping** doesn’t call `saveLayer()` (unless
  explicitly requested with `Clip.antiAliasWithSaveLayer`),
  so these operations aren’t as expensive as `Opacity`,
  but clipping is still costly, so use with caution.
  By default, clipping is disabled (`Clip.none`),
  so you must explicitly enable it when needed.
* To create a rectangle with rounded corners,
  instead of applying a clipping rectangle,
  consider using the `borderRadius` property offered
  by many of the widget classes.

---

## Implement grids and lists thoughtfully 

How your grids and lists are implemented
might be causing performance problems for your app.
This section describes an important best
practice when creating grids and lists,
and how to determine whether your app uses
excessive layout passes.

### Be lazy!

When building a large grid or list,
use the lazy builder methods, with callbacks.
That ensures that only the visible portion of the
screen is built at startup time.

For more information and examples, check out:

* [Working with long lists][] in the [Cookbook][]
* [Creating a `ListView` that loads one page at a time][]
  a community article by AbdulRahman AlHamali
* [`Listview.builder`][] API

[Cookbook]: {{site.url}}/cookbook
[Creating a `ListView` that loads one page at a time]: {{site.medium}}/saugo360/flutter-creating-a-listview-that-loads-one-page-at-a-time-c5c91b6fabd3
[`Listview.builder`]: {{site.api}}/flutter/widgets/ListView/ListView.builder.html
[Working with long lists]: {{site.url}}/cookbook/lists/long-lists

### Avoid intrinsics

For information on how intrinsic passes might be causing
problems with your grids and lists, see the next section.

---

## Minimize layout passes caused by intrinsic operations

If you've done much Flutter programming, you are
probably familiar with [how layout and constraints work][]
when creating your UI. You might even have memorized Flutter's
basic layout rule: **Constraints go down. Sizes go up.
Parent sets position.**

For some widgets, particularly grids and lists,
the layout process can be expensive.
Flutter strives to perform just one layout pass
over the widgets but, sometimes,
a second pass (called an _intrinsic pass_) is needed,
and that can slow performance.

### What is an intrinsic pass?

An intrinsic pass happens when, for example,
you want all cells to have the size
of the biggest or smallest cell (or some
similar calculation that requires polling all cells).

For example, consider a large grid of `Card`s.
A grid should have uniformly sized cells,
so the layout code performs a pass,
starting from the root of the grid (in the widget tree),
asking **each** card in the grid (not just the
visible cards) to return
its _intrinsic_ size&mdash;the size
that the widget prefers, assuming no constraints.
With this information,
the framework determines a uniform cell size,
and re-visits all grid cells a second time,
telling each card what size to use. 

### Debugging intrinsic passes

To determine whether you have excessive intrinsic passes,
enable the **[Track layouts option][]**
in DevTools (disabled by default),
and look at the app's [stack trace][]
to learn how many layout passes were performed.
Once you enable tracking, intrinsic timeline events
are labeled as '$runtimeType intrinsics'.

### Avoiding intrinsic passes

You have a couple options for avoiding the intrinsic pass:

* Set the cells to a fixed size up front.
* Choose a particular cell to be the
  "anchor" cell&mdash;all cells will be
  sized relative to this cell.
  Write a custom render object that
  positions the child anchor first and then lays
  out the other children around it.

To dive even deeper into how layout works,
check out the [layout and rendering][]
section in the [Flutter architectural overview][].


[Flutter architectural overview]: {{site.url}}/resources/architectural-overview
[how layout and constraints work]: {{site.url}}/development/ui/layout/constraints
[layout and rendering]: {{site.url}}/resources/architectural-overview#layout-and-rendering
[stack trace]: {{site.url}}/development/tools/devtools/cpu-profiler#flame-chart
[Track layouts option]: {{site.url}}/development/tools/devtools/performance#track-layouts

---

##  Build and display frames in 16ms

Since there are two separate threads for building
and rendering, you have 16ms for building,
and 16ms for rendering on a 60Hz display.
If latency is a concern,
build and display a frame in 16ms _or less_.
Note that means built in 8ms or less,
and rendered in 8ms or less,
for a total of 16ms or less.

If your frames are rendering in well under
16ms total in [profile mode][],
you likely don’t have to worry about performance
even if some performance pitfalls apply,
but you should still aim to build and
render a frame as fast as possible. Why?

* Lowering the frame render time below 16ms might not make a visual
  difference, but it **improves battery life** and thermal issues.
* It might run fine on your device, but consider performance for the
  lowest device you are targeting.
* As 120fps devices become more widely available,
  you’ll want to render frames in under 8ms (total)
  in order to provide the smoothest experience.

If you are wondering why 60fps leads to a smooth visual experience,
check out the video [Why 60fps?][]

[profile mode]: {{site.url}}/testing/build-modes#profile
[Why 60fps?]: {{site.youtube-site}}/watch?v=CaMTIgxCSqU

# Pitfalls

If you need to tune your app’s performance,
or perhaps the UI isn't as smooth as you expect,
the [DevTools Performance view][] can help!

Also, the Flutter plugin for your IDE might
be useful. In the Flutter Performance window,
enable the **Show widget rebuild information** check box.
This feature helps you detect when frames are
being rendered and displayed in more than 16ms.
Where possible,
the plugin provides a link to a relevant tip.

The following behaviors might negatively impact
your app's performance.

* Avoid using the `Opacity` widget,
  and particularly avoid it in an animation.
  Use `AnimatedOpacity` or `FadeInImage` instead.
  For more information, check out
  [Performance considerations for opacity animation][].

* When using an `AnimatedBuilder`,
  avoid putting a subtree in the builder
  function that builds widgets that don’t
  depend on the animation. This subtree is
  rebuilt for every tick of the animation.
  Instead, build that part of the subtree
  once and pass it as a child to
  the `AnimatedBuilder`. For more information,
  check out [Performance optimizations][].

* Avoid clipping in an animation.
  If possible, pre-clip the image before animating it.

* Avoid using constructors with a concrete `List`
  of children (such as `Column()` or `ListView()`)
  if most of the children are not visible
  on screen to avoid the build cost.

# Resources

For more performance info, check out the following resources:

* [Performance optimizations][] in the AnimatedBuilder API page
* [Performance considerations for opacity animation][]
  in the Opacity API page
* [Child elements' lifecycle][] and how to load them efficiently,
  in the ListView API page
* [Performance considerations][] of a `StatefulWidget`

[Child elements' lifecycle]: {{site.api}}/flutter/widgets/ListView-class.html#child-elements-lifecycle
[`CustomPainter`]: {{site.api}}/flutter/rendering/CustomPainter-class.html
[DevTools Performance view]: {{site.url}}/development/tools/devtools/performance
[Performance optimizations]: {{site.api}}/flutter/widgets/AnimatedBuilder-class.html#performance-optimizations
[Performance considerations for opacity animation]: {{site.api}}/flutter/widgets/Opacity-class.html#performance-considerations-for-opacity-animation
[`RenderObject`]: {{site.api}}/flutter/rendering/RenderObject-class.html