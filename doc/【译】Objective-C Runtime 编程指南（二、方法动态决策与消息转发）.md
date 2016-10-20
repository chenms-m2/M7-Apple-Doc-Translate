[Objective-C runtime Programming Guide 官档传送门](https://developer.apple.com/library/content/documentation/Cocoa/Conceptual/ObjCruntimeGuide/Introduction/Introduction.html#//apple_ref/doc/uid/TP40008048-CH1-SW1)

本文翻译自2009-10-19版的官档。
您可以在官档结尾的Document Revision History中查阅版本。
注：【】包含的部分引自原文。

## 动态方法决策【Dynamic Method Resolution】
本章描述了如何动态地提供方法实现。

### 动态方法决策
有时候，你想要动态的提供方法实现。例如，Objective-C(后文简称OC)中属性的@dynamic指令。
```
@dynamic propertyName;
```
这个指令告诉编译器我们会动态的提供属性相关的方法。

我们可以使用 resolveInstanceMethod: 和resolveClassMethod: 来动态地提供实例方法和类方法的实现。
OC的方法类似于一个至少包含两个参数（self、_cmd）的C函数，你可以使用class_addMethod函数来为类添加方法，例如，下面的函数：
```
void dynamicMethodIMP(id self, SEL _cmd) {
    // implementation ....
}
```
你可以像下面的代码一样，使用resolveInstanceMethod，动态地为类添加方法（本例中方法名为resolveThisMethodDynamically）：
```
@implementation MyClass
+ (BOOL)resolveInstanceMethod:(SEL)aSEL
{
    if (aSEL == @selector(resolveThisMethodDynamically)) {
          class_addMethod([self class], aSEL, (IMP) dynamicMethodIMP, "v@:");
          return YES;
    }
    return [super resolveInstanceMethod:aSEL];
}
@end
```

方法转发与方法动态决策一般来说是正交的（译注：互不依赖与干扰）【are, largely, orthogonal】，在方法转发机制介入之前，类会有一次机会进行方法动态决策。如果respondsToSelector: 或instancesRespondToSelector:方法被调用，动态决策机制会获取一次机会，看是否能提供给selector需要的IMP。如果你实现了resolveInstanceMethod: ，但某些selector想交给方法转发机制处理，那么这些selector你要返回NO。

### 动态加载
OC的程序可以在运行期动态加载和链接类或分类。这些代码可以无差别的合并到已加载的类或分类中。

动态加载可以处理很多问题，例如，系统偏好设置这个应用，可以动态地加载功能模块。

在Cocoa中，动态加载一般用于程序的定制化。其他人可以为你的应用创提供模块，你可以动态地加载使用，例如，IB可以加载自定义的调色盘，系统偏好设置可以加载自定义的配置模块。可加载模块扩展了应用的功能，你只需要决定是否许可加载，不需要关心加载模块的实现【but could not have anticipated or defined yourself】。你提供框架，其他人为你提供代码。

尽管runtime有相关的函数可以加载Mach-O文件中的OC模块（objc_loadModules, defined in objc/objc-load.h），但Cocoa的NSBundle类提供了更多更方便的接口，这些接口集成了相关的服务并且是面向对象的。你可以参考Foundation框架的NSBundle类的文档。

## 消息转发
给一个对象发送它不能处理的消息会产生错误，但抛出这个错误之前，runtime给了对象一次额外的机会来尝试处理消息。

### 转发 
如果对象无法处理接收到的消息，在抛出错误之前，runtime会先向对象发送forwardInvocation:消息，消息的唯一参数是NSInvocation对象，它封装了原始消息和相关的参数。
你可以实现forwardInvocation:方法，给消息一个默认的响应，或者用其他的方式避开错误。就像方法名中所暗示的，forwardInvocation:方法通常可以将消息转发给其他对象处理。

为了了解转发机制的范围和意图【To see the scope and intent of forwarding】，我们来看这样一个场景。假如你设计的类可以响应negotiate消息，你希望它的响应中包含另一个对象的响应，可以在它的方法体中，向另一个对象发送negotiate消息。

进一步想，假如你想要使用其他的对象对negotiate的响应作为自己对negotiate的响应。一种方式是你的类继承其他类，也就继承了它的negotiate。但这种方式并不总是合适，很多情况下，你的类不能继承自那个类。

如果你无法通过继承的方式实现，一个简单的方式就是直接持有其他类的对象，直接向其发送negotiate消息即可，如下：
```
- (id)negotiate
{
    if ( [someOtherObject respondsTo:@selector(negotiate)] )
        return [someOtherObject negotiate];
    return self;
}
```

这种方式有点麻烦，尤其是你想将很多消息转发给另一个对象时，更是如此。你需要实现一个方法处理所有需要转发的消息。甚至在写代码时，你还无法确定所有的情况。需要转发的消息集可能依赖运行期的一些事件，而且未来可能改变。【That set might depend on events at runtime, and it might change as new methods and classes are implemented in the future.】

而forwardInvocation:方法提供了一种点对点的解决方案【ad hoc solution】，而且它是动态的。它的工作机制如下：如果接收者对象无法响应消息（没有实现消息对应的方法），那么runtime会向其发送一个forwardInvocation:消息，所有继承自NSObject的类都继承了该方法，不过NSObject的实现只是简单调用了doesNotRecognizeSelector:。你可以重写forwardInvocation:方法来将消息转发给其他对象。

为了转发消息，forwardInvocation:需要做以下事情：
- 决定是否转发消息
- 如果需要转发，则转发原始消息及相关参数

转发需要调用invokeWithTarget:方法，如下所示：
```
- (void)forwardInvocation:(NSInvocation *)anInvocation
{
    if ([someOtherObject respondsToSelector:
            [anInvocation selector]])
        [anInvocation invokeWithTarget:someOtherObject];
    else
        [super forwardInvocation:anInvocation];
}
```

返回值将会返回给原始的调用者，支持任意类型的返回值，包括id、结构体、double等。

forwardInvocation:方法可以认为是未识别消息的派发中心，它将消息打包后派发给不同的接收者。它也可以作为一个中转站，可以将所有消息转给另一个对象。它可以将消息转发给另一个对象，也可以直接“吞掉”消息以避免异常或错误。它还可以给几个不同的消息返回同样的响应。forwardInvocation:要做什么依赖于你，不管怎么说，它将多个对象链接为一个转发链，极大的扩展了程序设计的思路。

> 注意：forwardInvocation:只在消息无法正常处理时才会调用，例如，你想把negotiate转发给另一个对象处理，那么你的对象不能实现negotiate方法。

更详细信息请参考Foundation的NSInvocation类。

### 转发和多继承
转发可以模拟继承，并可以在OC编程中模拟多继承。如图 5-1所示，当你将消息转发给另一个对象，看上去就像你的方法实现是从另一个对象继承的。
图 5-1 转发

![](image/forwarding.gif)

图中，Warrior的实例将negotiate消息转发给Diplomat的实例。看上去战士（Warrior）像外交（Diplomat）官一样可以谈判（negotiate）了。Warrior实例看上去可以响应negotiate消息，在实际需要时候它确实可以响应（虽然实际上是Diplomat实例来执行的）。

转发消息的对象看上去从类树的两个不同分支上“继承”了方法实现——它自己所在分支和接收转发消息的对象所在的分支。在上面的例子中，Warrior看上去既继承了自己的父类，又继承了Diplomat。

转发可以模拟多继承的作用，但两者是有很大不同的：多继承将多种不同的功能组合到一个对象中，它会造成庞大的、多种功能的对象。而转发机制，将泽富分散到了不同的对象中。转发机制将问题分解给一组小对象解决，关联对象等操作对调用方来说是透明的。

### 替代对象【Surrogate Objects】
转发除了可以模拟多继承，还可以用来开发轻量级对象来代表或“覆盖【cover】”实质对象【substantial Objects】（译注：实际执行业务的对象）。替代对象代表其他对象，会将收到的消息转给它代表的对象。
The Objective-C Programming Language 文档的Remote Messaging章节中提到的代理【proxy】就是这种替代对象。代理会管理本地消息转发给远程接收者的细节，来确保可以正确地通过网络拷贝与获取参数等。但代理不会处理更多事务，它不会像远程对象那样执行业务逻辑，它只是给远程对象一个本地引用，让本地代码可以接受其他应用的消息。

当然，还有很多情况会用到替代对象。例如，你有一个对象管理了大量数据——复杂的图片或者从硬盘文件中加载的内容。建立这样的对象比较耗时，因此你可能想要采用懒加载，比如首次用到时加载或者在系统资源空闲时加载。这种情况下，你需要一个占位对象，来保证应用的其他部分能正常工作（译注：例如其他对象想和你的大数据对象沟通）。这个对象可能有能力执行一些业务，比如响应其他对象对数据信息的询问，但通常来讲，它只是个占位对象，接到相关的消息时，转发给实质对象处理。当它的forwardInvocation:接到了需要转发给实质对象的消息时，它要保证实质对象已经建立完毕，如果没有的话，它会建立实质对象。所有的消息都通过它转发给实质对象，从程序的其他对象角度看，替代对象和实质对象是一样的。

### 转发和继承
- 尽管转发可以模拟继承，但NSObject不会混淆两者，respondsToSelector:和isKindOfClass:方法只关注继承树，而不关注转发链。
例如，Warrior对象被询问是否能响应negotiate消息：
```
if ( [aWarrior respondsToSelector:@selector(negotiate)] )
    ...
```

结果是NO，尽管它可以接收negotiate消息不产生错误，甚至可以响应该消息（通过转发给Diplomat处理），依然是NO。

多数情况下，NO就是我们想要的，但少数情况下，这不是我们想要的结果。例如我们通过转发实现替代对象，或者为类扩展功能，这种情况下，转发机制对调用方应该是透明的。如果你要达到这个目的，你需要重写respondsToSelector:或和isKindOfClass:方法，来包含你的转发逻辑，如下所示：
```
- (BOOL)respondsToSelector:(SEL)aSelector
{
    if ( [super respondsToSelector:aSelector] )
        return YES;
    else {
        /* Here, test whether the aSelector message can     *
         * be forwarded to another object and whether that  *
         * object can respond to it. Return YES if it can.  */
    }
    return NO;
}
```
除了respondsToSelector:和isKindOfClass:外，instancesRespondToSelector:也需要重写，如果你使用了协议，还要重写conformsToProtocol:。类似的，如果你转发了消息，还要重写methodSignatureForSelector:，来保证返回真正执行业务的方法。例如，如果你的对象会把消息转发给实质对象【surrogate】，你可以像下面这样实现：
```
- (NSMethodSignature*)methodSignatureForSelector:(SEL)selector
{
    NSMethodSignature* signature = [super methodSignatureForSelector:selector];
    if (!signature) {
       signature = [surrogate methodSignatureForSelector:selector];
    }
    return signature;
}
```

你可以考虑把这些转发代码提取成一个单独的方法，所有需要重写的方法（包括forwardInvocation:），都调用它。

> 注意：这是一个高级技术，只适用于其他方案解决不了的场景。它也不是继承的替代方案。如果你要使用这项技术，一定要确保你完全理解转发对象和接受转发对象的所有行为。

本章节提到的方法主要存在于Foundation的NSObject类中，其中invokeWithTarget:方法存在于Foundation的NSInvocation类中。





