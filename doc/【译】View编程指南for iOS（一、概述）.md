[View Programming Guide for iOS 官方文档传送门](https://developer.apple.com/library/content/documentation/WindowsViews/Conceptual/ViewPG_iPhoneOS/Introduction/Introduction.html#//apple_ref/doc/uid/TP40009503-CH1-SW2)
本文翻译自2014-09-17版的官档。
您可以在官档结尾的Document Revision History中查阅版本。
注：【】包含的部分引自原文。

--------
## 关于window和view
在iOS中，你使用window和view来展示内容。window本身没有可见的内容，它只为view们提供基本的容器。window区域中的各部分内容，由view们定义。例如，你可能使用view来展示图片、图片、形状或它们的组合。你可以使用view们组合成更复杂的view。

### 概述
每个应用至少有一个window和view以展示内容。UIKit等库提供了一些预定义的view，你可以直接使用。这些view从简单的按钮、标签到复杂的tableView、pickerView及scrollView都有。如果预定义的view不能满足你的需要，你还可以自定义view，你需要自己管理它的绘制和事件处理。

#### view管理的应用的可视内容
view是UIView或其子类的实例，它管理window中的一个矩形区域。view负责绘制内容、事件处理，以及子view的布局。绘制包含很多技术，例如Core Graphics、OpenGL ES，以及UIKit，你可以使用这些技术在矩形区域来绘制形状、图片以及文本。view可以响应自己矩形区域中触摸事件，你可以使用手势识别器，或直接处理触摸事件。在view树【hierarchy】中，父view负责设置子view们的尺寸和位置，当然，可以动态的设置。动态修改子view的能力让你的view更好的适应环境的改变，例如界面旋转或动画。

你可以将view想象成砖块【block】，你可以使用它们来搭建界面。通常，你不会使用一个view展示所有的内容，二是使用多个view构建一个view树。树中的每个view展示一部分内容，它们通常擅长展示特定的内容。例如，UIKit提供的view们，分别擅长展示图片、文本及其他类型的内容。

> 相关章节：[View and Window Architecture](https://developer.apple.com/library/content/documentation/WindowsViews/Conceptual/ViewPG_iPhoneOS/WindowsandViews/WindowsandViews.html#//apple_ref/doc/uid/TP40009503-CH2-SW1)、[Views](https://developer.apple.com/library/content/documentation/WindowsViews/Conceptual/ViewPG_iPhoneOS/CreatingViews/CreatingViews.html#//apple_ref/doc/uid/TP40009503-CH5-SW1)

#### window协调【Coordinate】view们的展示
window是UIWindow的示例，它负责应用用户界面的整体展示。window和view（以及它们所属【owning】的ViewController）配合工作，来管理可见view树的交互与调整。多数情况下，应用的window不会改变。window创建后，通常不会变化，只是改变view的显示。每个应用至少有一个window，来在设备的主屏幕上展示view。如果设备连接了外部设备，应用可以创建第二个window，在额外的设备上的展示内容。

> 相关章节：[Windows](https://developer.apple.com/library/content/documentation/WindowsViews/Conceptual/ViewPG_iPhoneOS/CreatingWindows/CreatingWindows.html#//apple_ref/doc/uid/TP40009503-CH4-SW1)

#### 动画给用户提供界面变化的反馈
动画可以把view树上的变化可视化的反馈给用户。系统定义了标准的动画，来呈现模态view，或切换不同的view。view的很多属性可以直接执行动画（译注，理解为支持动画）。例如，你可以改变view的透明度、位置、尺寸、背景色以及其他属性。如果你和view底层的layer对象交互，你还可以执行更多动画。

> 相关章节：[Animations](https://developer.apple.com/library/content/documentation/WindowsViews/Conceptual/ViewPG_iPhoneOS/AnimatingViews/AnimatingViews.html#//apple_ref/doc/uid/TP40009503-CH6-SW1)

#### IB【Interface Builder】的角色
IB是一个应用，你可以使用它图形化的构建和配置window及view。使用IB，你将view们聚合在一个nib文件中。nib文件是一种资源文件，它存储了view和其他对象的保存版本【freeze-dried version】。当你在运行期加载nib文件时，文件中存储的对象会重新构建成实际对象，也就是你的代码可以操作的对象。

IB很大程度上简化了创建界面的工作。由于iOS中同时支持了IB和nib文件，你可以很轻松的使用nib文件来设计应用界面。

更多使用IB的信息，请参考Interface Builder User Guide（链接已失效）文档。更多ViewController管理nib文件的信息，请参考[View Controller Programming Guide for iOS]()文档的“创建自定义的内容ViewController”章节。

### 参考
由于view的复杂性和灵活性，一篇文档很难覆盖它的所有信息。你可以通过阅读下述文档来完善相关知识：
- ViewController对于管理view是很重要的。ViewController管理一个view树中所有view，并帮助它们在屏幕上展示。更多信息，请参考[View Controller Programming Guide for iOS](https://developer.apple.com/library/content/featuredarticles/ViewControllerPGforiPhoneOS/index.html#//apple_ref/doc/uid/TP40007457)文档。
- view是手势和触摸事件的核心接收者。更多使用手势识别器和直接处理触摸事件的信息，请参考[Event Handling Guide for iOS](https://developer.apple.com/library/content/documentation/EventHandling/Conceptual/EventHandlingiPhoneOS/Introduction/Introduction.html#//apple_ref/doc/uid/TP40009541)文档。 
- 自定义view需要使用现有的绘制技术来渲染内容。更多绘制view的信息，请参考[Drawing and Printing Guide for iOS](https://developer.apple.com/library/content/documentation/2DDrawing/Conceptual/DrawingPrintingiOS/Introduction/Introduction.html#//apple_ref/doc/uid/TP40010156)文档。
- 如果标准的动画不能满足你的需要，你可以使用Core Animation动画。更多使用CA实现动画的信息，请参考[Core Animation Programming Guide](https://developer.apple.com/library/content/documentation/Cocoa/Conceptual/CoreAnimation_guide/Introduction/Introduction.html#//apple_ref/doc/uid/TP40004514)文档。



