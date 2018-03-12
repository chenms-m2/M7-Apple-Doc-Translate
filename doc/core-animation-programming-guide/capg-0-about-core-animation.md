# 关于Core Animation

[原文：Core Animation Programming Guide - About Core Animation](https://developer.apple.com/library/content/documentation/Cocoa/Conceptual/CoreAnimation_guide/Introduction/Introduction.html#//apple_ref/doc/uid/TP40004514-CH1-SW1)

Core Animation（后文简称CA）是一个iOS/OSX平台上的图形渲染与动画框架【infrastructure】，你可以用它来对view或其他可见的元素（layer？）作动画。绘制动画每一帧时的大部分工作，CA都会为你处理。你通常只需要配置动画参数（例如开始点和结束点），然后告诉CA开始动画就可以了。CA会将实际的绘制工作交给图形硬件去加速渲染。这种自动图形加速可以生成高帧率平滑的动画，而不会加重CPU负担，不会拖慢你的app。

如果你在开发iOS的app的话，虽然你没意识到，但你一直在使用CA。如果你在开发OSX的app，你只需要一点工作就可以使用CA。CA位于AppKit和UIKit的下层，它已经紧密的集成到了Cocoa和Cocoa Touch中view的流程【workflow】中。Core Animation also has interfaces that extend the capabilities exposed by your app’s views and give you more fine-grained control over your app’s animations。

![](image/ca_architecture_2x.png)

### 概览
你可能不会直接使用CA，如果你要使用的话，要清楚CA在你的app框架【infrastructure】中是什么角色。

##### CA管理app的内容
CA本身不是绘制系统。It is an infrastructure for compositing and manipulating your app’s content in hardware。CA框架的核心是layer，你使用layer来manage and manipulate内容。A layer captures your content into a bitmap that can be manipulated easily by the graphics hardware。大多数app中，layer为view管理内容，但你也可以根据需要创建独立的layer

相关章节：[Core Animation Basics](), [Setting Up Layer Objects]()

##### 修改layer会触发动画
你使用CA创建的动画，大多数都涉及到layer属性的修改。和view类似，layer有长方形bounds，有position，有opacity，有transform，还有很多其他可见的属性，你可以进行修改。对于大多数属性来说，修改时会创建一个从旧值到新值的隐式动画。如果你想更好的控制动画的行为，你可以对属性执行显式动画。

相关章节：[Animating Layer Content](), [Advanced Animation Tricks](), [Layer Style Property Animations](), [Animatable Properties]()

##### layer可以被组织成树【Hierarchies】
layer可以被组织成父子关系的树，layer在树中的位置对其显示效果的影响和view树是类似的。The hierarchy of a set of layers that are attached to views mirrors the corresponding view hierarchy。你也可以向layer树中添加独立的layer，在不添加view的情况下来扩展可视内容。

相关章节：[Building a Layer Hierarchy]()


##### action可以改变layer的默认行为
layer的隐式动画是通过action实现的，action是实现了某些预定义接口的对象。CA使用action实现了layer相关的默认动画。你可以使用自定义的action来实现自定义动画，或者执行其他自定义的行为。你可以为layer的某个属性设置action，当这个属性变化的时候，CA就会获取设置的action，并执行这个action。

相关章节：[Changing a Layer’s Default Behavior]()

### 如何使用本文档
本文档适用于想要更好的控制动画或想利用layer来提升绘制性能的开发者。本文档也提供了iOS与OSX中view和layer集成的相关信息。在iOS和OSX上，view和layer的集成是有区别的，理解了这些有助于创建更高性能的动画。

### 预备知识
你应该理解了目标平台（iOS/OSX）上view的架构，以及view动画的使用，如果你不具备这些知识，请参考下列文档：

- iOS：[View Programming Guide for iOS](https://developer.apple.com/library/content/documentation/WindowsViews/Conceptual/ViewPG_iPhoneOS/Introduction/Introduction.html#//apple_ref/doc/uid/TP40009503)
- OSX：[View Programming Guide](https://developer.apple.com/library/content/documentation/Cocoa/Conceptual/CocoaViewsGuide/Introduction/Introduction.html#//apple_ref/doc/uid/TP40002978)

