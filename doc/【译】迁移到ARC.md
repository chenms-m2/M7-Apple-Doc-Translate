[Transitioning to ARC Release Notes 官档传送门](https://developer.apple.com/library/content/releasenotes/ObjectiveC/RN-TransitioningToARC/Introduction/Introduction.html#//apple_ref/doc/uid/TP40011226)
本文翻译自2013-08-08版的官档。
您可以在官档结尾的Document Revision History中查阅版本。
注：【】包含的部分引自原文。

## 迁移到ARC【Transitioning to ARC Release Notes】
自动引用计数（ARC）是一项编译器功能，可以自动管理Objective-C对象的内存。你不用再去考虑retain、release操作，可以专注于业务代码，像对象图、程序中对象间的关系等。

![](image/ARC_Illustration.jpg)

### 摘要
ARC会在编译期添加代码，来确保对象的生命周期符合你的需求。从概念上看，ARC和MRC遵循同样的内存管理规则（详细可参考[Advanced Memory Management Programming Guide](https://developer.apple.com/library/content/documentation/Cocoa/Conceptual/MemoryMgmt/Articles/MemoryMgmt.html#//apple_ref/doc/uid/10000011i)），ARC会为你添加合适的内存管理语句。
为了让编译期生成正确的代码，ARC对方法的使用及自由桥接【toll-free bridging 】的使用做了约束。ARC还为对象及属性【property】引入新的修饰词。
ARC从Xcode 4.2开始支持，支持OS X v10.6及以上，iOS4及以上，但弱引用【Weak references】从OSX v10.7和iOS5开始支持。

Xcode提供了工具，可以帮你自动转化代码到ARC（如移除retain、release），也能协助你修复无法自动处理（点选Edit > Refactor > Convert to Objective-C ARC）的问题。迁移工具会将工程中所有文件转化为ARC。如果你需要再在某些文件上使用MRC，你可以选择只将部分文件转化为ARC。
也可以参考这两篇文档：
- [Advanced Memory Management Programming Guide](https://developer.apple.com/library/content/documentation/Cocoa/Conceptual/MemoryMgmt/Articles/MemoryMgmt.html#//apple_ref/doc/uid/10000011i)
- [Memory Management Programming Guide for Core Foundation](https://developer.apple.com/library/content/documentation/CoreFoundation/Conceptual/CFMemoryMgmt/CFMemoryMgmt.html#//apple_ref/doc/uid/10000127i)

### ARC概述
你不再需要关注什么时候插入retain、release以及autorelease，ARC会在编译期评估对象的生命周期，并自动地在合适的位置插入内存管理语句。编译器还会帮你生成合适的dealloc方法。【 if you’re only using ARC the traditional Cocoa naming conventions are important only if you need to interoperate with code that uses manual reference counting】

一个Person类的实现如下：
```
@interface Person : NSObject
@property NSString *firstName;
@property NSString *lastName;
@property NSNumber *yearOfBirth;
@property Person *spouse;
@end
 
@implementation Person
@end
```

（对象属性默认是strong的，strong的描述请参考ARC Introduces New Lifetime Qualifiers一节）
使用ARC，你可以像下面一样实现方法：

```
- (void)contrived {
    Person *aPerson = [[Person alloc] init];
    [aPerson setFirstName:@"William"];
    [aPerson setLastName:@"Dudney"];
    [aPerson setYearOfBirth:[[NSNumber alloc] initWithInteger:2011]];
    NSLog(@"aPerson: %@", aPerson);
}
```
ARC会处理内存管理，所以Person和NUNumber对象都不会内存泄露。
你可以像下面这样实现Person类的takeLastNameFrom:方法：
```
- (void)takeLastNameFrom:(Person *)person {
    NSString *oldLastname = [self lastName];
    [self setLastName:[person lastName]];
    NSLog(@"Lastname changed from %@ to %@", oldLastname, [self lastName]);
}
ARC ensures that oldLastName is not deallocated before the NSLog statement.
```
ARC会确保NSLog执行时oldLastName尚未销毁。

### ARC实行新规则
ARC增加了一些新规则，以此来提供更加可靠的内存管理。这些规则可以让代码遵循最佳实践，或者是简化代码（你不需要再手动管理内存）。如果违反了这些规则，你会立即得到编译期错误，减少了运行期的bug。
- 你不能调用dealloc，也不能重写或调用retain、release、autorelease和retainCount。当然也不能使用类似@selector(retain), @selector(release)等等。
你可以重写dealloc，但在dealloc你不能释放实例变量，而仅仅应该管理资源，例如调用系统类或MRC代码中类似[systemClassInstance setDelegate:nil] 的方法。
重写dealloc时不要调用[super dealloc]，编译器会自动处理。
如果你使用了Core Foundation及相关库的类（可参考 Managing Toll-Free Bridging一节），你仍然可以使用CFRetain, CFRelease语句。
- 你不能使用NSAllocateObject和NSDeallocateObject。
你使用alloc创建对象，runtime会负责销毁对象。
- 你不能在C结构体中使用对象指针。你应该创建类来封装数据，代替结构体。
- id和void *指针不能随意互相转型。
你需要使用特殊的转型方式来告知编译器对象的生命周期。Objective-C对象 和 Core Foundation类型需要特殊转型（详细可参考 Managing Toll-Free Bridging一节）。
- 你不能使用NSAutoreleasePool。 
ARC提供了@autoreleasepool，它比NSAutoreleasePool性能更好。
- 你不能使用内存域【memory zones】。
没有必要使用NSZone了，现代的Objective-C运行时已经忽略了它。

为了和MRC代码交互，ARC对方法命名做了一定的约束。
- 你不能使用new作为访问器方法的前缀，如果你想用new作为一个属性的前缀，那么它的getter方法不能以new为前缀，如下所示：
```
// Won't work:
@property NSString *newTitle;
 
// Works:
@property (getter=theNewTitle) NSString *newTitle;
```

### ARC引入了新的生命周期修饰词
ARC引入了新的生命周期修饰词，并引入了弱引用【weak references】，弱引用不会影响对象的生命周期，如果对象没有了强引用，弱引用会自动指向nil。
你应该使用这些修饰词来管理应用的对象图，实践中，ARC无法避免循环引用，但使用弱引用帮你处理这个问题。

#### 属性的特性【Property Attributes】
ARC新引入了strong和weak特性，如下所示：
```
// The following declaration is a synonym for: @property(retain) MyClass *myObject;
@property(strong) MyClass *myObject;
 
// The following declaration is similar to "@property(assign) MyClass *myObject;"
// except that if the MyClass instance is deallocated,
// the property value is set to nil instead of remaining as a dangling pointer.
@property(weak) MyClass *myObject;
```
ARC中，对象默认是strong的。

#### 变量修饰词【Variable Qualifiers】
你可以使用下面的生命周期修饰词来修饰变量。
```
__strong
__weak
__unsafe_unretained
__autoreleasing
```
- __strong是默认值，对象有强引用时会保持存在。
- __weak不会保持对象存在，当对象没有强引用时，原本指向它的弱引用变量会自动指向nil。
- __unsafe_unretained类似 __weak，不会保持对象存在，不同的是，当对象没有强引用时,  __unsafe_unretaine变量不会执行nil，而是成为了野指针。
- __autoreleasing用于按引用传输的参数（id *），或作为方法返回值的autorelease对象。

用修饰词修饰变量时，正确的格式如下：
```
ClassName * qualifier variableName;
```
例如：
```
MyClass * __weak myWeakReference;
MyClass * __unsafe_unretained myUnsafeReference;
```
其他的变体从技术角度讲是不对的，但编译器会忽略它【Other variants are technically incorrect but are “forgiven” by the compiler】，为了理解这个问题，请参考[http://cdecl.org/](http://cdecl.org/)。

在栈【stack】内（译注：我理解为方法体内）使用__weak时要注意，如下面的例子：
```
NSString * __weak string = [[NSString alloc] initWithFormat:@"First Name: %@", [self firstName]];
NSLog(@"string: %@", string);
```
string赋值后，随后就会被NSLog用到，但因为没有强引用指向它，赋值后，string指向的对象会被立即销毁。NSLog打印时，string已经是nil。这种情况下，编译器会给你警告。
传引用参数时，你也需要注意，下述代码是可以的：
```
NSError *error;
BOOL OK = [myObject performOperationWithError:&error];
if (!OK) {
    // Report the error.
    // ...
```
然而，error的声明实际是这样的：
```
NSError * __strong e;
```
而方法声明一般是这样的：
```
-(BOOL)performOperationWithError:(NSError * __autoreleasing *)error;
```
因此编译器要重写此处代码：
```
NSError * __strong error;
NSError * __autoreleasing tmp = error;
BOOL OK = [myObject performOperationWithError:&tmp];
error = tmp;
if (!OK) {
    // Report the error.
    // ...
```
__strong和 __autoreleasing的不匹配导致编译器创建了临时变量，你可以将方法参数声明为id __strong * 以接收__strong变量地址，或者将变量声明为 __autoreleasing。

#### 使用生命周期修饰词来避免循环引用
你可以使用生命周期修饰词来避免循环引用。例如，你的对象图是一个树状结构，父对象需要引用子对象，子对象也要引用父对象，你可以让父对象强引用子对象，而让子对象弱引用父对象。但有些情况就不这么显而易见了，比如涉及到block时。
在MRC时代， __block id x;有个效果是x不会被block retain，而在ARC时代，这么写x依然会被retain，正确的写法是 __unsafe_unretained __block id x。但 __unsafe_unretained 如同其名字所示，它不会被retain，但同时也是不安全的，变量可能会成为野指针，所以这种写法是不推荐的。所以更好的选择是使用 __weak，或者将  __block修饰的变量适时置nil，以打破循环引用。

下面的代码展示了MRC时代时有问题的写法【The following code fragment illustrates this issue using a pattern that is sometimes used in manual reference counting】
```
MyViewController *myController = [[MyViewController alloc] init…];
// ...
myController.completionHandler =  ^(NSInteger result) {
   [myController dismissViewControllerAnimated:YES completion:nil];
};
[self presentViewController:myController animated:YES completion:^{
   [myController release];
}];
```
ARC时代，我们可以这样处理，使用__block，并在completionHandler中将myController置nil。
```
MyViewController * __block myController = [[MyViewController alloc] init…];
// ...
myController.completionHandler =  ^(NSInteger result) {
    [myController dismissViewControllerAnimated:YES completion:nil];
    myController = nil;
};
```
或者，我们可以使用一个__weak的临时变量，如下面所示：
```
MyViewController *myController = [[MyViewController alloc] init…];
// ...
MyViewController * __weak weakMyViewController = myController;
myController.completionHandler =  ^(NSInteger result) {
    [weakMyViewController dismissViewControllerAnimated:YES completion:nil];
};
```
对于有意义的循环引用【non-trivial cycles】（译注：block中的逻辑依赖引用的对象），你可以这样做：
```
MyViewController *myController = [[MyViewController alloc] init…];
// ...
MyViewController * __weak weakMyController = myController;
myController.completionHandler =  ^(NSInteger result) {
    MyViewController *strongMyController = weakMyController;
    if (strongMyController) {
        // ...
        [strongMyController dismissViewControllerAnimated:YES completion:nil];
        // ...
    }
    else {
        // Probably nothing...
    }
};
```
如果你的工程不兼容 __weak（例如iOS4及以下），你可以使用 __unsafe_unretained，但对于有意义的循环引用【nontrivial cycles】，使用__unsafe_unretained是不合适的，因为你很难确定__unsafe_unretained 指向的对象是否还合法。

### ARC使用新的语句管理自动引用池
使用ARC，你不能再手动生成NSAutoreleasePool对象，而应该使用@autoreleasepool block。
```
@autoreleasepool {
     // Code, such as a loop that creates a large number of temporary objects.
}
```
这个简单的结构让编译器可以检查【reason about】引用计数状态。进入block时，会将一个自动释放池压栈，正常退出block时（如break、return、goto以及fall-through等），自动释放池被从栈中弹出。为了兼容现有的代码，异常退出block时不会弹栈。
这个语法在所有Objective-C模式中都存在，它比NSAutoreleasePool性能好，我们鼓励使用它来替换所有的NSAutoreleasePool。

### Outlets管理模式成为跨平台一致的【Patterns for Managing Outlets Become Consistent Across Platforms】
ARC时代Outlets管理模式有所改变，并且在iOS和OS X平台上保持了一致性。管理模式通常如下：outlets通常为weak，例外的是从File’s Owner 到nib或storyboard的top-level objects的outlets，它们为strong。（译注：这一句直译，但对nib不够了解，尚未完全理解）。
更详细的描述请参考[Resource Programming Guide的Nib Files一章](https://developer.apple.com/library/content/documentation/Cocoa/Conceptual/LoadingResources/CocoaNibs/CocoaNibs.html#//apple_ref/doc/uid/10000051i-CH4)

### 栈变量（局部变量）会自动初始化为nil
在ARC中，strong、weak以及autorelease的栈变量会隐式初始化为nil，如下：
```
- (void)myMethod {
    NSString *name;
    NSLog(@"name: %@", name);
}
```
log会打印出null，而不是崩溃。

### 使用编译器flag来开关ARC
你可以使用编译器flag -fobjc-arc来打开ARC，如果你的工程中混杂了ARC和MRC文件，你可以使用此flag打开指定文件的ARC。如果你的工程默认是ARC的，你可以用-fno-objc-arc来关闭特定文件的ARC。
Xcode4.2之后，OS X v10.6及以上，iOS4及以上，才支持ARC。其中，OS X v10.7及以上，iOS5及以上，才支持弱引用。

### 管理自由桥接【Toll-Free Bridging】
很多Cocoa的应用中，你需要使用Core Foundation风格的对象。这些对象可能来自Core Foundation（如 CFArrayRef、CFMutableDictionaryRef等），也可能来自符合Core Foundation管理的库，如Core Graphics（你可能使用里面的 CGColorSpaceRef、CGGradientRef等）。
编译器不会自动管理Core Foundation对象的生命周期，你必须按照Core Foundation的内存管理规则来调用CFRetain、CFRelease（或和特定类型相关的变体）。（更详细的信息参见[Memory Management Programming Guide for Core Foundation](https://developer.apple.com/library/content/documentation/CoreFoundation/Conceptual/CFMemoryMgmt/CFMemoryMgmt.html#//apple_ref/doc/uid/10000127i)）
如果你需要再Objective-C对象和 Core Foundation风格对象间转型，你需要告诉编译器该对象的拥有关系。你可以使用转型修饰词（定义在objc/runtime.h中），或使用Core Foundation风格的宏（定义在NSObject.h中）。

- __bridge 转换对象，但不转换拥有关系。
- __bridge_retained 或 CFBridgingRetain，将Objective-C对象转换为Core Foundation对象，同时将拥有关系也转换给你。
你有责任调用CFRelease或相关的方法来解除拥有关系。
- __bridge_transfer 或 CFBridgingRelease，将对象转换为Core Foundation Objective-C对象，将拥有关系转给ARC。
ARC有责任解除拥有关系。

例如，你有下述的MRC代码：
```
- (void)logFirstNameOfPerson:(ABRecordRef)person {
 
    NSString *name = (NSString *)ABRecordCopyValue(person, kABPersonFirstNameProperty);
    NSLog(@"Person's first name: %@", name);
    [name release];
}
```
你可以转化为下面的ARC代码：
```
- (void)logFirstNameOfPerson:(ABRecordRef)person {
 
    NSString *name = (NSString *)CFBridgingRelease(ABRecordCopyValue(person, kABPersonFirstNameProperty));
    NSLog(@"Person's first name: %@", name);
}
```

### 编译器能处理Cocoa方法返回的CF对象
如果方法按照惯例命名，那么即使返回了CF对象，编译器也可以理解，并知道如何处理。例如，在iOS中，UIColor的CGColor方法返回了CGColor对象，编译器通过方法名，可以知道调用方不会拥有该对象。但涉及到Cocoa对象和CF对象，你仍然要做类型转换，下面是一个例子：
```
NSMutableArray *colors = [NSMutableArray arrayWithObject:(id)[[UIColor darkGrayColor] CGColor]];
[colors addObject:(id)[[UIColor lightGrayColor] CGColor]];
```
### 转型函数参数时可以使用拥有关系的关键字
如果你在函数调用时需要转型Objective-C对象和CF对象，你需要告诉编译器对象的拥有关系信息。CF对象的内存管理规则可参考[Memory Management Programming Guide for Core Foundation](https://developer.apple.com/library/content/documentation/CoreFoundation/Conceptual/CFMemoryMgmt/CFMemoryMgmt.html#//apple_ref/doc/uid/10000127i)，Objective-C对象的内存管理规则可参考[Advanced Memory Management Programming Guide](https://developer.apple.com/library/content/documentation/Cocoa/Conceptual/MemoryMgmt/Articles/MemoryMgmt.html#//apple_ref/doc/uid/10000011i)。
下面的代码中，传给CGGradientCreateWithColors函数的array需要转型，arrayWithObjects: 返回的array对象拥有关系不传给该函数，因此参数使用了__bridge修饰词。
```
NSArray *colors = <#An array of colors#>;
CGGradientRef gradient = CGGradientCreateWithColors(colorSpace, (__bridge CFArrayRef)colors, locations);
```
下面的代码中，根据CF的内存管理规则，使用了相应的CF管理函数：
```
- (void)drawRect:(CGRect)rect {
    CGContextRef ctx = UIGraphicsGetCurrentContext();
    CGColorSpaceRef colorSpace = CGColorSpaceCreateDeviceGray();
    CGFloat locations[2] = {0.0, 1.0};
    NSMutableArray *colors = [NSMutableArray arrayWithObject:(id)[[UIColor darkGrayColor] CGColor]];
    [colors addObject:(id)[[UIColor lightGrayColor] CGColor]];
    CGGradientRef gradient = CGGradientCreateWithColors(colorSpace, (__bridge CFArrayRef)colors, locations);
    CGColorSpaceRelease(colorSpace);  // Release owned Core Foundation object.
    CGPoint startPoint = CGPointMake(0.0, 0.0);
    CGPoint endPoint = CGPointMake(CGRectGetMaxX(self.bounds), CGRectGetMaxY(self.bounds));
    CGContextDrawLinearGradient(ctx, gradient, startPoint, endPoint,
                                kCGGradientDrawsBeforeStartLocation | kCGGradientDrawsAfterEndLocation);
    CGGradientRelease(gradient);  // Release owned Core Foundation object.
}
```
### 转换工程时的共通问题
当工程迁移到ARC时，你可能会遇到很多问题，下面是一些共通的问题及解决方案。
- 你不能调用retain、release以及autorelease。
你也**不能**这样写：

```
while ([x retainCount]) { [x release]; }
```

- 你不能调用dealloc。
如果你想实现单例，或想替换掉init中的对象，你可能想调用dealloc（译注：没完全明白这两种用法中为什么想dealloc）。单例的话，你应该使用共享实例模式，init方法中，你不需要调用dealloc，当你重写self时，对象会自动释放【the object will be freed when you overwrite self】。
- 你不能使用NSAutoreleasePool对象。
使用@autoreleasepool{}来代替。 这会让你的自动释放池有一个blcok结构体【This forces a block structure on your autorelease pool】,@autoreleasepool{}比NSAutoreleasePool快6倍。由于@autoreleasepool如此之快，它可以替换掉很多曾经的“性能黑科技【performance hacks】”。
迁移者【migrator】可以处理简单的NSAutoreleasePool使用，但无法处理复杂的情况，或者是在@autoreleasepool体中定义变量，在@autoreleasepool外使用的情况。

- 在init方法中，ARC需要你将[super init]的结果赋值给self。
在ARC中，下面的写法是**非法**的：
```
[super init];
```
简单修复后的版本：
```
self = [super init];
```
更好的修复是这样做，在继续之前检查返回值是否为nil：
```
self = [super init];
if (self) {
   ...
```
- 你不能实现自定义的retain、release。
实现自定义的retain、release或破坏弱引用机制。可能有以下几种场景，让你想实现自定义的方法：
    - 性能。
    NSObject的retain、release已经非常快了，如果你认为有问题，可以报告bug【If you still find problems, please file bugs】。
    - 实现自定义的弱引用。
    请用__weak。
    -  实现单例类。
    请使用共享实例模式，或者直接使用类方法，这样连实例都不用创建。
    
- 赋值【“Assigned”】给实例变量会是强引用    
MRC时代，实例变量是无拥有关系的引用【non-owning references】，也就是说把对象赋值给给实例变量，不会影响该对象的生命周期。实现一个强引用的属性，你需要按照内存管理规则来实现或让编译器合成相应的访问器方法。MRC时代，如果你想实现一个弱引用的属性，可以像下面这样做：
```
@interface MyClass : Superclass {
    id thing; // Weak reference.
}
// ...
@end
 
@implementation MyClass
- (id)thing {
    return thing;
}
- (void)setThing:(id)newThing {
    thing = newThing;
}
// ...
@end
```

ARC时代，实例变量默认是强引用的。你将对象赋值给实例变量，它会被强引用，生命周期会受到影响。迁移工具无法确定哪些实例变量应该是弱引用，你需要显式的使用__weak实例变量或属性，如下：
```
@interface MyClass : Superclass {
    id __weak thing;
}
// ...
@end
 
@implementation MyClass
- (id)thing {
    return thing;
}
- (void)setThing:(id)newThing {
    thing = newThing;
}
// ...
@end
```
或
```
@interface MyClass : Superclass
@property (weak) id thing;
// ...
@end
 
@implementation MyClass
@synthesize thing;
// ...
@end
```
- 你不能在C结构体中使用强引用的id变量。
下面代码是**编译不过**：
```
struct X { id x; float y; };
```
因为x默认是强引用，但编译器无法合成足够的代码以保持其正确性。例如，你通过一段代码为结构体赋值，这段代码结束时会执行清理操作【free】，那么每个id指向的对象都可能被销毁，而这很可能发生在结构体被销毁之前。编译器无法完善的处理这种情况，所以在ARC中是禁止C结构体中包含强引用的id变量。有如下几种替代方案：
1. 使用Objective-C对象代替C结构体。
这也是最佳实践方案。
2. 如果使用Objective-C对象是次佳的方案，比如你在一个array存储巨量的结构体，那么你可以使用void *替代。
这需要你显式的转型，后文还有相关讨论。
3. 使用__unsafe_unretained来修饰对象。
这种方法可能对下面这种【semi-common】的模式有用：
```
struct x { NSString *S;  int X; } StaticArray[] = {
  @"foo", 42,
  @"bar, 97,
...
};
```
结构体声明如下：
```
struct x { NSString * __unsafe_unretained S; int X; }
```
这种方式不太安全，对象有可能被释放，但对于字符串常量这种不用担心被释放的场景，可能是有用的。
- 你不能直接转型id和void *（包括CF对象），详情可参考Managing Toll-Free Bridging一节。

### FAQ
Q：**我应该怎么思考ARC，它是怎么处理的retain和release【How do I think about ARC? Where does it put the retains/releases?】**
A：不要去想retain/release的调用问题，多想想你的应用的业务逻辑【algorithms】，多想想强引用、弱引用，多想想对象间的拥有关系，以及可能出现的循环引用。

Q：**我还需要写dealloc吗**
A：可能需要。
ARC不能自动处理malloc/free，也不能自动管理Core Foundation对象的生命周期，以及文件描述符等等，你可能需要再dealloc自己处理这些问题。
你不需要释放实例变量，但你可能要在系统类或者非ARC编译的代码中的类上调用[self setDelegate:nil]。
ARC中是禁止在dealloc中调用[super dealloc]的，runtime会自动完成这些工作。

Q：**ARC中还有循环引用问题吗**
A：有。
ARC会自动的retain/release，因此也继承了MRC的循环引用问题。但迁移代码时，基本不会有内存泄露，因为你类中的属性已经声明了是否需要retain。

Q：**ARC中block怎么工作**
A：ARC下，当你将block在栈中传递【pass blocks up the stack】时（例如作为方法的返回值），可以正常工作，你不再需要copy。
唯一有问题的是ARC中NSString * __block myString的变量会被retain，你需要替换为 __block NSString * __unsafe_unretained myString，或者 __block NSString * __weak myString（这种更推荐）。

Q：**我能在Snow Leopard版本的Mac上开发OS X上的ARC应用吗**
A：不能。Snow Leopard的Xcode4.2不包含10.7 SDK，它只支持iOS ARC应用开发。而Lion的Xcode4.2既支持iOS也支持OS X了。这意味这你必须再Lion系统上构建ARC应用，但可以在Snow Leopard上运行。

Q：**ARC下，我能创建包含retained pointers的C数组吗**
A：可以，如下所示：
```
// Note calloc() to get zero-filled memory.
__strong SomeClass **dynamicArray = (__strong SomeClass **)calloc(entries, sizeof(SomeClass *));
for (int i = 0; i < entries; i++) {
     dynamicArray[i] = [[SomeClass alloc] init];
}
 
// When you're done, set each entry to nil to tell ARC to release the object.
for (int i = 0; i < entries; i++) {
     dynamicArray[i] = nil;
}
free(dynamicArray);
```
这有几点要注意：
- 你需要 __strong SomeClass **  来转型，因为默认为 __autoreleasing SomeClass **。
- 创建的内存必须用0填充。
- 释放C数组钱，你必须将所有元素指针指向nil（不能用memset或bzero）。
- 你应该避免使用memcpy和realloc。

Q：**ARC慢吗**
A：取决于你怎么衡量，不过通常来看，答案是“不慢”。
编译器移除了很多没必要的retain/release，并投入很多努力来加速runtime。典型地，ARC下，常用的“返回retain/autoreleased对象”模式更快了，实际上并不一定要把对象放入自动释放池。
在调试模式时【common debug configurations】并没有进行优化，因此你在-O0时会比-Os时看到更多的retain/release。

Q：**ARC在ObjC++模式也能工作吗**
A：可以。你甚至可以把strong/weak id引用的对象放到类或容器中。ARC会在赋值构造器、析构器等时自动合成retain/release逻辑，来保证正常工作。

Q：**哪些类不支持弱引用**
A：当前有以下类：
NSATSTypesetter, NSColorSpace, NSFont, NSMenuView, NSParagraphStyle, NSSimpleHorizontalTypesetter, 以及 NSTextView.
> 注意，在OS X v10.7中，额外地有下述的类不支持弱引用，NSFontManager, NSFontPanel, NSImage, NSTableCellView, NSViewController, NSWindow, NSWindowController以及AV Foundation framework库中所有的类。
对于属性来说，你可以用assign代替weak，对于变量来说，你可以用__unsafe_unretained代替 __weak。
额外地，ARC下你不能创建以下类的弱引用，NSHashTable, NSMapTable, 以及 NSPointerArray。

Q：**我在继承NSCell或其他使用了NSCopyObject的类时，应该注意什么**
A：没什么可注意的。ARC考虑到了这些之前必须显式retain的情况，ARC中，所有的copy方法都应该只copy实例变量【ARC takes care of cases where you had to previously add extra retains explicitly. With ARC, all copy methods should just copy over the instance variable】。（译注：没完全理解）

Q：**我能对特定文件禁用ARC吗**
A：可以。
当你迁移到ARC时，默认所有源文件都会设置编译选项 -fobjc-arc。你可以对特定文件设置编译选项 -fno-objc-arc，来禁用该文件的ARC。
Xcode中，在the target Build Phases tab，打开Compile Sources group展开文件列表，找到你要设置的文件，双击，在弹窗里填写-fno-objc-arc，点击完成就可以了。

![](image/fno-objc-arc.png)

Q：**Mac上的垃圾回收机制（GC）会被废弃吗**
A：GC在OS X Mountain Lion v10.8版本已被废弃，后续的OS X上将移除该机制。ARC是替代技术。
Xcode 4.3及以后的版本中有迁移工具，可以将支持GC的Mac程序迁移到ARC。

> 注意：Mac App Store 中的app，应该尽早完成GC到ARC的迁移，因为 Mac App Store的审核机制会拒绝使用废弃技术的App。











