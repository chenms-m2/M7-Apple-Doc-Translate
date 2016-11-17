[View Controller Programming Guide for iOS 官方文档传送门](https://developer.apple.com/library/ios/featuredarticles/ViewControllerPGforiPhoneOS/index.html#//apple_ref/doc/uid/TP40007457)
本文翻译自2015-09-16版的官档。
您可以在官档结尾的Document Revision History中查阅版本。
注：【】包含的部分引自原文。

--------

## 定义ViewController

### 定义子类
你通过自定义UIViewController的子类来展示应用的内容。大部分自定义ViewController都是内容ViewController，它拥有自己所有的view，并管理view需要的数据。相反，容器ViewController不会拥有所有view，有些view由它的子ViewController管理。两者的定义步骤大多数是类似的。下面的章节会讨论到。

对于内容ViewController来说，最常见的父类如下：
- UITableViewController。如果你的ViewController的主view是个列表，可以使用它。
- UICollectionViewController。如果你的ViewController的是个集合（如网格），可以使用它。
- 其他情况使用UIViewController。

对于容器ViewController来说，父类是什么取决于你想修改现有的容器ViewController，还是想完全自己创建。如果要修改现有的，选择你想修改的容器。如果想自己创建，通常是继承UIViewController。更多关于创建容器的信息，请参考后文的“实现容器ViewController”一节。

#### 定义UI
使用Xcode的storyboard文件来可视化的定义ViewController的UI。尽管你可以使用代码创建UI，但storyboard是个更好的方式，使用storyboard可以使ViewController的内容可视化，并可以为不同的环境【environments】定制不同的view树。可视化的创建UI可以让你快速的修改UI，并能让你在不构建运行App的情况下查看效果。

图4-1展示storyboard的示例。每个矩形区域展示了一个ViewController和它的view们。ViewController间的箭头是他们的relationships和segues。Relationship连接容器ViewController和它的子ViewController，Segue实现在ViewController间导航。

图4-1 包含一组ViewController的storyboard

![](_image/【译】View Controller 编程指南（二·一、定义ViewController之定义子类）/storyboard_bird_sightings_2x.png)

每个新工程都有一个主storyboard，它通常已经包含一个或多个VIewController。你可以直接从库里将新ViewController拖到画布上。新ViewController默认没有关联的类，你必须在Xcode的Identity inspector中为它指定类。

使用storyboard编辑器来进行下述操作：
- 为ViewController添加、排列以及配置view。
- 连接outlet及action，参考后文“处理用户交互”一节。
- 为ViewController创建relationships and segues，参考后文“使用Segue一节”。
- 为不同的尺寸类别【size class】定制布局和view，参考后文“构建可适配的界面”
- 添加手势识别器来处理用户交互，参考[Event Handling Guide for iOS](https://developer.apple.com/library/content/documentation/EventHandling/Conceptual/EventHandlingiPhoneOS/Introduction/Introduction.html#//apple_ref/doc/uid/TP40009541)文档。

如果你没使用过storyboard，你可以在Start Developing iOS Apps Today文档（译注：链接已失效）中找到一步步教你入门的教程。

#### 处理用户交互
应用的响应对象负责对接收到的事件进行合适的处理。尽管ViewController是响应对象，但它一般不会直接处理touch事件。ViewController一般通过下述几种方式处理事件。
- ViewController定义action方法来处理更高级的事件。action方法一般会响应下述action：
    - 特定【Specific】action。Control组件和部分view会调用action来报告特定的交互。
    - 手势识别器。手势识别器会调用action方法来报告手势的当前状态。使用ViewController来处理手势的状态变化及完成。
- ViewController监听系统或其他对象发出的通知。通知会报告变化，ViewController可能据此改变自己的状态。
- ViewController作为其他对象的数据源【data source】或代理【delegate】。ViewController经常为tableView何collectionView管理数据。它也可以作为其他对象的代理，例如作为CLLocationManager对象的代理，可以收到定位的坐标值。

响应事件通常会更新view的内容，因此需要持有这些view的引用。ViewController是个适合为view定义outlet的地方。使用列表4-1中的语法来将outlet声明属性。示例中的自定义类定义了两个outlet（使用IBOutlet关键词标识），并定义了一个action方法（使用IBAction关键词标识）。outlet存储了button和textField的引用，action方法会响应button的点击事件。

表4-1 在ViewController类中定义outlet和action
```
// OBJECTIVE-C
@interface MyViewController : UIViewController
@property (weak, nonatomic) IBOutlet UIButton *myButton;
@property (weak, nonatomic) IBOutlet UITextField *myTextField;
 
- (IBAction)myButtonAction:(id)sender;
 
@end


// SWIFT
class MyViewController: UIViewController {
    @IBOutlet weak var myButton : UIButton!
    @IBOutlet weak var myTextField : UITextField!
    
    @IBAction func myButtonAction(sender: id)
}

```
在storyboard中，记得将outlet及action和相关的view连接。连接它们可以保证view被加载时会自动完成配置。更多在IB（Interface Builder）中创建outlet及action的信息，请参考Interface Builder Connections Help文档（译注：链接已失效）。更多事件处理的信息，请参考[Event Handling Guide for iOS](https://developer.apple.com/library/content/documentation/EventHandling/Conceptual/EventHandlingiPhoneOS/Introduction/Introduction.html#//apple_ref/doc/uid/TP40009541)文档。

#### 在运行期展示view
storyboard使得加载和展示ViewController的view非常简单。UIKit会在需要的时候自动从storyboard中加载view。作为加载过程的一部分，UIKit会执行下述操作：
1. 使用storyboard中的信息初始化view。
2. 连接outlet及action。
3. 为ViewController的view属性时设置根view。
4. 调用ViewController的awakeFromNib方法。
当这个方法被调用时，ViewController的特性集合【trait collection】还是空的，view不一定在最终位置上。
5. 调用ViewController的viewDidLoad方法。
使用这个方法添加或移除view，修改布局约束，以及为view加载数据。

在将ViewController的view显示到屏幕之前，UIKit提供了额外的机会来配置view，在显示到屏幕后，也有机会来配置view。典型地，UIKit会执行下述操作：
1. 调用ViewController的viewWillAppear: 来通知view即将显示到屏幕上。
2. 更新view的布局。
3. 将view显示到屏幕上。
4. view显示到屏幕上后调用viewDidAppear:。

当你添加、删除或修改view的尺寸或位置时，记得为view添加或移除相关约束。进行布局相关的修改会导致UIKit将布局标记为dirty。下一个更新循环（译注：应该指Runloop循环）时，布局引擎会使用当前约束来计算view的尺寸和位置，并使用结果修改view树。

更多使用storyboard创建view的信息，请参考[UIViewController Class Reference]()文档。

#### 管理view布局
当view的尺寸和位置变化时，UIKit会更新view树的布局信息。对于使用自动布局配置的view，UIKit使用自动布局引擎来根据当前约束来更新布局。UIKit还会通知可能相关的对象，例如被呈现ViewController【presentation controller】，它可以对布局的变化做适当的响应。

处理布局的过程中，UIKit会特定的阶段发出通知，接到通知后你可以进行额外的布局处理，你可以修改布局约束，或在约束生效后微调布局。UIKit会在处理过程进行操作：
1. 如果需要的话，更新ViewController和它的view的特性集合【trait collections】。请参考[When Do Trait and Size Changes Happen](https://developer.apple.com/library/content/featuredarticles/ViewControllerPGforiPhoneOS/TheAdaptiveModel.html#//apple_ref/doc/uid/TP40007457-CH19-SW6)章节。
2. 调用ViewController的viewWillLayoutSubviews方法。
3. 调用当前UIPresentationController对象的containerViewWillLayoutSubviews方法。
4. 调用ViewController的根view的layoutSubviews方法。
默认的实现会使用当前约束计算布局信息，然后遍历view树，调用每个子view的layoutSubviews方法。
5. 将计算的布局信息应用在view上。
6. 调用ViewController的viewDidLayoutSubviews方法。
7. 当前UIPresentationController对象的containerViewDidLayoutSubviews方法。

ViewController可以使用viewWillLayoutSubviews和viewDidLayoutSubviews方法来这行额外的操作，这些操作可能会影响布局过程。在布局前，你可以添加或删除view、更新view的尺寸或位置、更新约束，或更新其他view相关的属性。布局之后，你可以重新加载tableView的数据、更新其他view的内容，或对view的尺寸或位置做最后的调整。

有一些技巧可以帮你更有效的管理布局：
- 使用自动布局。使用自动布局的约束，在适配不同尺寸的屏幕时，更加灵活和简单。
- 利用顶部和底部的布局guide。以布局guide为参照可以确保你的内容始终课件。顶部布局guide会适应状态栏和导航条。类似的，底部不guide可以适应tabbar或工具栏。
- 添加或移除view时要更新约束。如果你动态的添加或移除view，记得更新相关的约束。
- ViewController的view们执行动画时，临时移除约束。使用UIKit Core Animation为view执行动画前，先移除view的约束，动画执行完毕后，再添加回来。如果动画结束后view的尺寸或位置改变了，记得更新约束。

更多关于呈现ViewController的信息，以及呈现ViewController在ViewController架构中的角色，请参考[The Presentation and Transition Process](https://developer.apple.com/library/content/featuredarticles/ViewControllerPGforiPhoneOS/PresentingaViewController.html#//apple_ref/doc/uid/TP40007457-CH14-SW7)章节。

#### 有效的管理内存
大多数内存分配由你来决定，表4-1列举了UIViewController中哪些地方适合分配或回收内存。大多数回收内存涉及移除对象的强引用，将属性或变量指向nil就可以移除对象的强引用。

表4-1 分配或回收内存的位置
- 初始化方法：创建ViewController需要的核心【critical】数据结构。
你的自定义初始化方法（init或类似的方法）有责任将ViewController的对象都设置为一个合适的初始状态。如果数据结构要确保能正确的操作，那么在初始化方法中创建它。
- viewDidLoad：创建或加载view要显示的数据。
使用viewDidLoad方法来加载你想显示的数据。这个方法被调用时，你的view可以被确保已经存在，并且处于一个合法状态。
- didReceiveMemoryWarning：响应内存不足通知。
使用这个发方法来销毁ViewController相关的非核心对象，尽量多的释放对象。
- dealloc：释放ViewController相关的核心数据结构。
重写这个方法来执行ViewController销毁前操作。系统会自动释放实例变量和属性中保存的对象，你不需要显式的释放它们。
