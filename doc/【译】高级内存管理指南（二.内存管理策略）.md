[Advanced Memory Management Programming Guide 官档传送门](https://developer.apple.com/library/content/documentation/Cocoa/Conceptual/MemoryMgmt/Articles/MemoryMgmt.html#//apple_ref/doc/uid/10000011-SW1)

本文翻译自2012-07-17版的官档。
您可以在官档结尾的Document Revision History中查阅版本。
注：【】包含的部分引自原文。

## 内存管理策略
内存管理的基本模型是引用计数，它基本上是用NSObject中定义的相关方案及标准的方法命名惯例组合而成的。NSObject还定义了一个dealloc方法，它会在对象被销毁时自动被调用。本文描述了正确的内存管理的所有基本规则，以及一些正确使用场景的示例。

### 基本内存管理规则
内存管理模型基于对象间关系。所有对象都有一个或多个拥有者，只要有拥有者，它就存在。如果对象没有了拥有者，runtime会自动的销毁它。你何时会拥有一个对象呢，Cocoa制定了下面的策略：
- 你创建，你拥有
你使用带有alloc、new、copy、mutableCopy前缀的方法（例如alloc, newObject, or mutableCopy）会创建对象。
- 你可以用retain来拥有对象
对象在其被访问的方法中，应该要保证是合法的【A received object is normally guaranteed to remain valid within the method it was received in】，并且方法也应该安全地将对象返回给调用者。你一般在下面两种情况下使用retain：（1）在init方法及property的setter方法中使用，建立拥有关系，来保存你需要的对象。（2）避免你需要的对象被其他的操作置为非法状态（详细描述请参考 避免你使用的对象被销毁 【Avoid Causing Deallocation of Objects You’re Using】一章）。
- 你不再需要对象时，要放弃对它的拥有
你可以给对象发送release或autorelease来放弃对它的拥有。在Cocoa的术语中，放弃对一个对象的拥有，可以叫做释放【releasing】对象。
- 你不能释放一个你没拥有的对象
这是上述那些策略的推论。

### 一个简单的示例
下面的代码段，可以简单展示上述的策略
```
{
    Person *aPerson = [[Person alloc] init];
    // ...
    NSString *name = aPerson.fullName;
    // ...
    [aPerson release];
}
```
Person对象是被alloc方法创建的，所以后续不再需要时，对其发送了release消息。而person的name并非拥有型方法（alloc、new等）创建，因此也没有发送release【The person’s name is not retrieved using any of the owning methods, so it is not sent a release message】。注意，此处使用了release，没有用autorelease。

### 使用autorelease来延迟释放
当你想要延迟发送release消息时，你可以使用autorelease，典型场景就是方法返回对象。例如，你可以像下述这样实现fullName方法。
```
- (NSString *)fullName {
    NSString *string = [[[NSString alloc] initWithFormat:@"%@ %@",
                                          self.firstName, self.lastName] autorelease];
    return string;
}
```
你拥有使用alloc方法创建的string，按照基本规则，在你失去对齐引用之前，你应该放弃对其的拥有。如果你使用release，那么对象会在方法返回之前就被释放（方法会返回一个非法对象）。使用autorelease，你暗示你要放弃对string的拥有，但允许调用方在string被销毁前使用它。

你也可以这样实现fullName方法：
```
- (NSString *)fullName {
    NSString *string = [NSString stringWithFormat:@"%@ %@",
                                 self.firstName, self.lastName];
    return string;
}
```
按照基本规则，你不会拥有stringWithFormat:方法创建的对象，因此你可以安全返回string对象。

为了做个对比，*下面这种写法是错误的*
```
- (NSString *)fullName {
    NSString *string = [[NSString alloc] initWithFormat:@"%@ %@",
                                         self.firstName, self.lastName];
    return string;
}
```
按照命名惯例，调用方不会拥有string对象，因此它没有理由发送release消息，于是就产生了内存泄露。

### 你不会拥有通过引用参数返回的对象
Cocoa的某些方法中指定了某些参数作为引用参数（参数类型一般为ClassName \*\* 或 id \*）。一个常见的模式是在出现问题时返回一个NSError的引用参数，例如NSData的initWithContentsOfURL:options:error:方法，或者NSString的initWithContentsOfFile:encoding:error: 方法。
这种情况下，也遵循上面提到的策略，当你执行这些方法时，你并没有创建NSError对象，所以你不会拥有它，也不需要对它发送release。参考下面的示例代码：
```
NSString *fileName = <#Get a file name#>;
NSError *error;
NSString *string = [[NSString alloc] initWithContentsOfFile:fileName
                        encoding:NSUTF8StringEncoding error:&error];
if (string == nil) {
    // Deal with error...
}
// ...
[string release];
```

### 实现dealloc来放弃对象的拥有关系
NSObject类定义了dealloc方法，它会在对象没有拥有者，内存可以回收（Cocoa的术语为释放freed或销毁deallocated）时自动调用。dealloc方法的角色是释放对象占用的内存及其他资源，包括其成员变量。

下面的示例展示了如何实现Person的dealloc方法：
```
@interface Person : NSObject
@property (retain) NSString *firstName;
@property (retain) NSString *lastName;
@property (assign, readonly) NSString *fullName;
@end
 
@implementation Person
// ...
- (void)dealloc
    [_firstName release];
    [_lastName release];
    [super dealloc];
}
@end
```

> 重要：永远不要直接调用其它对象的dealloc方法。
你必须再dealloc方法的末尾调用父类的实现。
你不应该把系统资源绑定到对象的生命周期。详细请参考 不要使用dealloc管理系统稀缺资源【Don’t Use dealloc to Manage Scarce Resources】一章。
应用被关闭时，对象的dealloc可能并未被调用。进程终止时内存会被自动清理，因此让操作系统管理资源比我们使用内存管理方法更加简单有效。

### Core Foundation 使用类似但是不同的规则
Core Foundation对象有类似的内存管理规则（详细可参考[Memory Management Programming Guide for Core Foundation](Memory Management Programming Guide for Core Foundation)）。Cocoa 和 Core Foundation的命名管理有所不同。Core Foundation的创建规则（参见[The Create Rule](https://developer.apple.com/library/content/documentation/CoreFoundation/Conceptual/CFMemoryMgmt/Concepts/Ownership.html#//apple_ref/doc/uid/20001148-103029)）不适用于返回Objective-C对象。因此，下面代码中，你不需要释放myInstance：

```
MyClass *myInstance = [MyClass createInstance];
```


