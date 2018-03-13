# Animatable Properties

Many of the properties in CALayer and CIFilter can be animated. This appendix lists those properties, along with the animation used by default.

### CALayer Animatable Properties

Table B-1 lists the properties of the CALayer class that you might consider animating. For each property, the table also lists the type of default animation object that is created to execute an implicit animation.

Table B-1  Layer properties and their default animations

- anchorPoint

Uses the default implied CABasicAnimation object, described in Table B-2.

- backgroundColor

Uses the default implied CABasicAnimation object, described in Table B-2.

- backgroundFilters

Uses the default implied CATransition object, described in Table B-3. Sub-properties of the filters are animated using the default implied CABasicAnimation object, described in Table B-2.

- borderColor

Uses the default implied CABasicAnimation object, described in Table B-2.

- borderWidth

Uses the default implied CABasicAnimation object, described in Table B-2.

- bounds

Uses the default implied CABasicAnimation object, described in Table B-2.

- compositingFilter

Uses the default implied CATransition object, described in Table B-3. Sub-properties of the filters are animated using the default implied CABasicAnimation object, described in Table B-2.

- contents

Uses the default implied CABasicAnimation object, described in Table B-2.

- contentsRect

Uses the default implied CABasicAnimation object, described in Table B-2.

- cornerRadius

Uses the default implied CABasicAnimation object, described in Table B-2.

- doubleSided

There is no default implied animation.

- filters

Uses the default implied CABasicAnimation object, described in Table B-2. Sub-properties of the filters are animated using the default implied CABasicAnimation object, described in Table B-2.

- frame

**This property is not animatable. You can achieve the same results by animating the bounds and position properties.**

- hidden

Uses the default implied CABasicAnimation object, described in Table B-2.

- mask

Uses the default implied CABasicAnimation object, described in Table B-2.

- masksToBounds

Uses the default implied CABasicAnimation object, described in Table B-2.

- opacity

Uses the default implied CABasicAnimation object, described in Table B-2.

- position

Uses the default implied CABasicAnimation object, described in Table B-2.

- shadowColor

Uses the default implied CABasicAnimation object, described in Table B-2.

- shadowOffset

Uses the default implied CABasicAnimation object, described in Table B-2.

- shadowOpacity

Uses the default implied CABasicAnimation object, described in Table B-2.

- shadowPath

Uses the default implied CABasicAnimation object, described in Table B-2.

- shadowRadius

Uses the default implied CABasicAnimation object, described in Table B-2.

- sublayers

Uses the default implied CABasicAnimation object, described in Table B-2.

- sublayerTransform

Uses the default implied CABasicAnimation object, described in Table B-2.

- transform

Uses the default implied CABasicAnimation object, described in Table B-2.

- zPosition


Uses the default implied CABasicAnimation object, described in Table B-2.


Table B-2 lists the animation attributes for the default property-based animations.

Table B-2  Default Implied Basic Animation

- Class

CABasicAnimation

- Duration

0.25 seconds, or the duration of the current transaction

- Key path

Set to the property name of the layer.

Table B-3 lists the animation object configuration for default transition-based animations.

Table B-3  Default Implied Transition

- Class

CATransition

- Duration

0.25 seconds, or the duration of the current transaction

- Type

Fade (kCATransitionFade)

- Start progress

0.0

- End progress

1.0

### CIFilter Animatable Properties

Core Animation adds the following animatable properties to Core Image’s CIFilter class. These properties are available only on OS X.

- name
- enabled

For more information about these additions, see CIFilter Core Animation Additions.