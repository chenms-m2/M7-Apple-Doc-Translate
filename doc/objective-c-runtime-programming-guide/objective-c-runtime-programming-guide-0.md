[Objective-C runtime Programming Guide 官档传送门](https://developer.apple.com/library/content/documentation/Cocoa/Conceptual/ObjCruntimeGuide/Introduction/Introduction.html#//apple_ref/doc/uid/TP40008048-CH1-SW1)

本文翻译自2009-10-19版的官档。
您可以在官档结尾的Document Revision History中查阅版本。
注：【】包含的部分引自原文。

## 简介
Objective-C（下文简称OC）可以延迟做决定，将做决定的时机从编译期、链接期延迟到了运行期。只要可能的话，它就会动态地处理问题。这也意味着OC不仅需要编译器，还需要一个runtime系统（下文简称runtime）来执行编译完的代码。runtime扮演着OC的操作系统的角色【The runtime system acts as a kind of operating system for the Objective-C language】，它保证了OC可以正常工作。
这篇文档侧重于【looks at】NSObject类以及OC程序如何与runtime系统交互。典型地，它会检查类的动态加载，以及对象间的消息转发。它还可以提供运行期间对象的信息。
通过阅读这篇文档，你可以理解runtime是如何工作的，以及你能如何利用它。如果你只是开发App，没有太大必要阅读本文档。

### 文档组织
本文档有以下章节
- [runtime版本和平台【runtime Versions and Platforms】](https://developer.apple.com/library/content/documentation/Cocoa/Conceptual/ObjCruntimeGuide/Articles/ocrtVersionsPlatforms.html#//apple_ref/doc/uid/TP40008048-CH106-SW1)
- [与runtime交互【Interacting with the runtime】](https://developer.apple.com/library/content/documentation/Cocoa/Conceptual/ObjCruntimeGuide/Articles/ocrtInteracting.html#//apple_ref/doc/uid/TP40008048-CH103-SW1)
- [消息发送【Messaging】](https://developer.apple.com/library/content/documentation/Cocoa/Conceptual/ObjCruntimeGuide/Articles/ocrtHowMessagingWorks.html#//apple_ref/doc/uid/TP40008048-CH104-SW1)
- [动态方法决策【Dynamic Method Resolution】](https://developer.apple.com/library/content/documentation/Cocoa/Conceptual/ObjCruntimeGuide/Articles/ocrtDynamicResolution.html#//apple_ref/doc/uid/TP40008048-CH102-SW1)
- [消息转发【Message Forwarding】](https://developer.apple.com/library/content/documentation/Cocoa/Conceptual/ObjCruntimeGuide/Articles/ocrtForwarding.html#//apple_ref/doc/uid/TP40008048-CH105-SW1)
- [类型编码【Type Encodings】](https://developer.apple.com/library/content/documentation/Cocoa/Conceptual/ObjCruntimeGuide/Articles/ocrtTypeEncodings.html#//apple_ref/doc/uid/TP40008048-CH100-SW1)
- [属性【Declared Properties】](https://developer.apple.com/library/content/documentation/Cocoa/Conceptual/ObjCruntimeGuide/Articles/ocrtPropertyIntrospection.html#//apple_ref/doc/uid/TP40008048-CH101-SW1)

### 其他参考
[Objective-C runtime Reference](https://developer.apple.com/reference/objectivec/1657527-objective_c_runtime)描述了runtime中的数据结构和函数，你的程序可以使用这些API和runtime交互。例如，你可以添加类或方法，或者获取所有已加载的类的信息。
[Programming with Objective-C](https://developer.apple.com/library/content/documentation/Cocoa/Conceptual/ProgrammingWithObjectiveC/Introduction/Introduction.html#//apple_ref/doc/uid/TP40011210)描述了OC语言。
[Objective-C Release Notes](https://developer.apple.com/library/content/releasenotes/Cocoa/RN-ObjectiveC/index.html#//apple_ref/doc/uid/TP40004309)描述了近期OS X发布版本中runtime的变化。

## runtime版本和平台
不同的平台上，runtime有不同的版本。
### 早期【Legacy】版本和现代版本
runtime有早期和现代两种版本，现代版本是OC 2.0中引入的，相比于早起版本，它有一些新功能。现代版本的API可以从[Objective-C runtime Reference](https://developer.apple.com/reference/objectivec/1657527-objective_c_runtime)查看。

最值得注意的新功能，是新版本中实例变量是“非脆弱的”。
- 早期版本中，如果你改变了类的实例变量布局【layout of instance variables】 ，需要重新编译【recompile】其子类。
- 现代版本中，你则不需要重新编辑子类。

另外，现代版本支持属性对应的实例变量的自动合成。

### 平台
iOS App，OS X v10.5及之后的64位程序，均采用现代版本。

## 与runtime交互
OC和runtime的交互有3个级别：通过OC代码、通过NSObject的方法、直接调用runtime API。

### OC代码
大多数情况下，runtime会自动在幕后完成工作，你只需要编写及编译OC代码即可。
当你编译包含OC的类和方法的代码时，编译器会自动创建OC动态特性所需要的数据结构与函数。数据结构捕获类、分类以及协议中的信息；包含类和协议对象（可参考 Objective-C Programming Language文档）、方法的selecor、实例变量模板，以及其他从源码中抽【distilled】出的信息。最重要的runtime函数是消息发送的函数，它会被源码中发送消息的方法调用。

### NSObject方法
Cocoa中大多数类是NSObject的子类，继承了它的方法（特例是NSProxy类，参考 Message Forwarding 章节）。它的方法确立了行为，对象或类对象从它的方法中继承了相关行为。有时候，NSObject只定义了行为的模板，但不会提供具体实现的代码。

例如，NSObject定义了description方法来返回对象的信息。这个方法主要用于GDB打印对象。NSObject不清楚其子类会有什么信息，它的实现是打印对象类型和内存地址，其子类可以重写方法来返回更详细的信息。例如，NSArray重写了这个方法，来打印自己包含的对象的列表。

NSObject的一些方法只是简单从runtime查询信息，这类方法可用于反射【introspection】。例如，class方法可查看自己的类，isKindOfClass:与isMemberOfClass:可以查看对象在继承树中的位置，respondsToSelector:可以检查对象能否响应特定的消息，conformsToProtocol:可以检查对象是否实现了协议要求的方法，methodForSelector:可以查看的selector对应的Method。类似的，Method也可以反射自己的信息。

### runtime函数
runtime是一个动态库，它的数据结构与函数的接口声明在/usr/include/objc中的头文件中，很多函数可以让你使用C来实现一些类似编译器对OC代码的处理。其他的基础功能由NSObject的方法提供【Others form the basis for functionality exported through the methods of the NSObject class】。这些函数可以扩展runtime的接口，或者生成一些工具来增强开发环境；在常规的OC开发中不会使用。当然，在OC开发中的某些时机，使用runtime函数是很有帮助的。相关函数请参考[Objective-C Runtime Reference](https://developer.apple.com/reference/objectivec/1657527-objective_c_runtime).

## 消息发送
本章描述了方法调用【message expressions】如何转化成了objc_msgSend函数调用，以及如何通过名字引用方法。之后介绍了objc_msgSend可以给你什么帮助，以及在需要的时候，如何规避动态绑定。

### objc_msgSend函数
在OC中，消息直到运行期才会和方法实现绑定，例如下面的方法调用：
```
[receiver message]
```
编译器会将其转化为objc_msgSend函数调用。这个函数调用会把接收者【receiver】和方法的selector作为首要的参数：
```
objc_msgSend(receiver, selector)
```
在方法调用中传入的参数，objc_msgSend也会处理：
```
objc_msgSend(receiver, selector, arg1, arg2, ...)
```
消息发送函数会处理动态绑定：
- 它查找selector对应的方法实现【procedure (method implementation) 】，由于同名方法可能被多个类实现，它会根据接收者的类型，查找到相应的方法实现。
- 然后它调用方法实现，并将接收者指针及其他参数传入。
- 最后它将方法实现的返回值作为自己的返回值返回。

> 编译器会生成消息发送函数的代码，你**不应该**在代码直接调用。

消息发送机制的关键在于编译器为每个类和对象创建的结构，每个类的结构中有两个基本要素：
- 一个指向superclass的指针
- 一个类的派发表【dispatch table】，表中包含selector和其方法实现的映射。例如，setOrigin::的selector关联到setOrigin::的方法实现，display的selector关联到display的方法实现。
当一个新对象创建的时候，系统会为其开辟内存，并初始化其实例变量，在所有变量之前，是一个指向对象的class的指针。这个指针名为isa，通过它，对象可以访问自己的class，并进一步访问父类及更高层的祖先类。

> 注意： 不严格的说，对象必须有isa指针来和runtime交互。对象必须是struct objc_object（定义在objc/objc.h），一般来说，你不需要创建自己的根对象，继承自NSObject和NSProxy的类会自动拥有isa指针。

这些类和对象的关系如图3-1所示：

图 3-1 消息发送

![](image/messaging1.gif)

当对象接到方法调用的消息时，消息发送函数根据isa指针找到类，并在类的派发表中查找selector对应的方法实现。如果没有查找到，函数会根据superclass指针找到父类，并在父类中查找。查找失败函数就会沿着类树向上找，直到到达NSObject类。如果查找成功，函数会调用selector对应的方法实现，并把接收者作为参数传入。
这就是在运行期选择方法实现的实现方法，或者用面对对象的术语说，方法动态绑定到了消息。
为了加速消息发送的进程，runtime缓存了selector和曾经使用的方法实现的映射。每个类都有自己的缓存，它能缓存selector和祖先类中的方法实现映射。在查找派发表之前，消息发送系统会先检查接收者的缓存（理论上说，用过的方法实现，不久会再用到）。如果缓存命中，则消息发送几乎和函数调用一样快。一旦程序运行时间足够长，缓存有了充足的“热身”【warm up】，那么几乎所有的查找都能在缓存中完成。随着程序的运行，缓存也会不断添加新的映射。

### 使用隐藏参数
objc_msgSend函数查找到selector的方法实现时，除了把方法调用传入的参数全部传给方法实现外，还会额外传入两个隐藏参数：
- 接收者对象
- 方法的selector
这些参数向方法实现提供了方法调用的全部信息，之所以说“隐藏参数”，是因为源代码的方法定义中并没有这两个参数，而是在编译期插入了方法实现。
尽管隐藏参数不是显式声明的，方法中仍然可以使用这两个参数（使用方式和其他实例变量没有区别）。方法中使用self引用接收者，使用_cms引用selector。下面的示例中，_cmd引用了strange方法的selector，self引用了strange消息的接收者。
```
- strange
{
    id  target = getTheReceiver();
    SEL method = getTheMethod();
 
    if ( target == self || method == _cmd )
        return nil;
    return [target performSelector:method];
}
```
self参数很有用，事实上，这也是方法体中可以使用实例变量的原因。

### 获取方法实现的地址【Getting a Method Address】
如果你想绕开动态绑定，唯一的方式是获取到方法实现的地址，然后直接调用它，像函数一样。这在极少数的情况下是有用的，例如一个方法需要多次调用，你使用这种方式避开每次方法调用的开销。
使用NSObject类中定义了methodForSelector:方法，你能获取到指向方法实现的指针，使用这个指针可以直接执行方法实现。使用methodForSelector:时，要小心的转型为合适的类型，包括参数和返回值的类型。
下面的代码展示了直接执行setFilled:方法的方法实现。
```
void (*setter)(id, SEL, BOOL);
int i;
 
setter = (void (*)(id, SEL, BOOL))[target
    methodForSelector:@selector(setFilled:)];
for ( i = 0 ; i < 1000 ; i++ )
    setter(targetList[i], @selector(setFilled:), YES);
```
前两个参数是接收者（self）和selector（_cmd），这些参数在方法签名中是隐藏的，但如果使用函数方式执行方法实现时，需要显式的传入这两个参数。
使用methodForSelector来绕开动态绑定机制，能节省一些消息发送的时间，但这种用法只应该用在方法需要多次调用的场景中，如上面代码中的for循环中。
注意，methodForSelector是Cocoa的runtime提供的功能，而不是OC语言提供的。





