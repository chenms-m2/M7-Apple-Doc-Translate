[Advanced Memory Management Programming Guide 官档传送门](https://developer.apple.com/library/content/documentation/Cocoa/Conceptual/MemoryMgmt/Articles/MemoryMgmt.html#//apple_ref/doc/uid/10000011-SW1)

本文翻译自2012-07-17版的官档。
您可以在官档结尾的Document Revision History中查阅版本。
注：【】包含的部分引自原文。

## 实践中的内存管理
在内存管理策略一章中的基本原则简单而明确，但还有一些实用的方法可以让我们的内存管理更简单，也让我们的程序更可靠、更健壮，并且占用更少的资源。

### 使用访问器方法【Accessor】简化内存管理
如果你的类有一个属性是对象类型，你应该确保在使用它的过程中，它不会被销毁。因此在它被赋值的时候，你需要声明对新值的拥有关系，并放弃对旧值的拥有关系。
看上去有点乏味，但如果你坚持使用访问器方法，内存管理问题会显著地减少。如果你还是使用对实例变量retain、release的方式，不可避免的会出现问题。

假设有一个Counter对象，你想要给它赋值。
```
@interface Counter : NSObject
@property (nonatomic, retain) NSNumber *count;
@end;
```
属性【property】声明了两个访问器方法，通常你应该让编译器来自动合成访问器方法。不过看一看这些方法可能是怎么实现的，也是有帮助的。
比如，在getter方法中，你直接返回自动生成的实例变量就可以了，不涉及到retain和release：
```
- (NSNumber *)count {
    return _count;
}
```
而在setter方法中，可能有其他人在再使用新值，因此你需要假设新值在任意时刻都可能被销毁，你需要通过retain新值来避免这个问题。对于旧值，你需要发送release来放弃对它的拥有关系（在Objective-C中，对nil发送消息不会有问题，因此_count变量还没被赋过值的话，也没有问题）。你需要先retain，再release，如果旧值和新值是同一个对象时，先release可能导致对象直接被销毁，再retain已经没有意义。

```
- (void)setCount:(NSNumber *)newCount {
    [newCount retain];
    [_count release];
    // Make the new assignment.
    _count = newCount;
}
```
### 使用访问器方法设置属性值
假设你想要实现一个方法来重置count的值，你有两种选择，第一种是用alloc创建NSNumber实例，你需要调用release，以保持引用计数平衡。
```
- (void)reset {
    NSNumber *zero = [[NSNumber alloc] initWithInteger:0];
    [self setCount:zero];
    [zero release];
}
```

第二种方式使用便利构造方法【convenience constructor】来创建NSNumber实例，这种方式无需调用release。
```
- (void)reset {
    NSNumber *zero = [NSNumber numberWithInteger:0];
    [self setCount:zero];
}
```
注意两种情况下，我们赋值都使用了setter方法。
在简单的场景下，下面这种方式也能正确的赋值，它避开了访问器方法，但它在一些场景下可能导致内存泄露（如忘了retain或release，甚至实例变量的内存管理方式改变）。

```
- (void)reset {
    NSNumber *zero = [[NSNumber alloc] initWithInteger:0];
    [_count release];
    _count = zero;
}
```
注意，上面这种直接给变量赋值的方式，是不兼容KVO的。

### 不要在初始化方法及dealloc方法中使用访问器方法
唯一你不应该使用访问器方法的地方，就是初始化方法及dealloc方法中。
例如，你要将count初始化为0，可以这些实现init方法：
```
- init {
    self = [super init];
    if (self) {
        _count = [[NSNumber alloc] initWithInteger:0];
    }
    return self;
}
```
如果想要用非零值初始化count，可以实现一个initWithCount:方法：
```
- initWithCount:(NSNumber *)startingCount {
    self = [super init];
    if (self) {
        _count = [startingCount copy];
    }
    return self;
}
```
既然Counter类有一个对象类型的实例变量，那么它应该实现dealloc方法，来放弃对变量的拥有关系，并且在方法的最后调用父类的实现：
```
- (void)dealloc {
    [_count release];
    [super dealloc];
}

```

### 使用弱引用来避免循环引用
- 持有一个对象会创建一个对其的强引用，被强引用着的对象不会被销毁，直到它不再被强引用。循环引用，就是两个对象分别持有对彼此的强引用（可能是直接地，也可能是多个对象在互相引用时，形成了闭环，引用链上的某个对象，又持有了对第一个对象的强引用）。
图1 所示的对象关系图展示了潜在的循环引用。文档对象【Document】对每一页都有一个页对象【Page】，每个页对象都有一个属性来记录自己属于哪个文档对象。如果文档对页持有强引用，并且页对文档也持有强引用，那么两者都无法销毁。因为只有页被释放了，文档引用计数才会变为0从而被销毁，而页也只有在文档被销毁时才会被释放。

图1 潜在的引用循环

![](image/retaincycles_2x.png)

解决方法是采用弱【weak】引用。弱引用是一种非持有关系，对象弱引用其他对象时，不会持有其他对象。
为了维持对象图的正确性，我们还需要强【strong】引用（如果只存在弱引用，那上图中的文档和页等对象都没有拥有关系，则会被直接销毁）。Cocoa的惯例是，“父对象”持有对“子对象”的强引用，而“子对象”持有对“父对象”的弱引用。
因此，在图1中，文档对象持有对页对象的强引用，而页对象持有对文档对象的弱引用。
Cocoa中弱引用例子很多，包含但不限于，tableView的data source，outline view，notification的observer，及其它的target或delegate。

如果你只持有一个对象的弱引用，那么对它发送消息时要多加注意。如果你向被销毁的对象发送消息，程序会崩溃（译者注：这是MRR的文档，弱引用应该是概念上的，实践中应该类似属性的assign，ARC时候的弱引用，才有weak关键字，不会再导致崩溃）。你需要良好的设计来确保对象的合法性。大多数情况下，对象知道其他对象弱引用自己以避免循环引用，对象在被销毁时有责任通知弱引用自己的对象。例如，当对象注册到notification center时，会被center弱引用。当对象要被销毁时，它应该向center注销自己，以避免自己被销毁后center还给自己发消息。类似的，对象作为其他对象的delegate时，销毁时应该将delegate置nil，一般在自己的dealloc执行相关操作。

### 避免正在使用的对象被销毁
Cocoa的拥有关系策略可知，在方法体内对象是合法的，而且对象在被方法返回时也不会立即释放。getter方法返回实例变量或计算值时也不会有问题。我们所担心的是，在使用对象的时候，对象是否还合法。
确实有些特殊情况，主要是以下两类：
1. 对象被容器移除时。
```
heisenObject = [array objectAtIndex:n];
[array removeObjectAtIndex:n];
// heisenObject could now be invalid.
```
对象被容器移除时，会受到一个release消息（不是autorelease），如果容器是对象的唯一拥有者，对象release后会立即销毁，即本例中的heisenObject。

2. “父对象”销毁时。
```
id parent = <#create a parent object#>;
// ...
heisenObject = [parent child] ;
[parent release]; // Or, for example: self.parent = nil;
// heisenObject could now be invalid.
```
有时候你获取了一个对象，但它的“父对象”被直接或间接的销毁了，如果父对象是对象的唯一拥有者，那么对象也会直接被销毁（通常父对象dealloc方法中，对子对象执行了release，而不是autorelease）。

为了避免这些情况，你应该在获取对象后对其retain，而在使用完毕后，对其release。
```
heisenObject = [[array objectAtIndex:n] retain];
[array removeObjectAtIndex:n];
// Use heisenObject...
[heisenObject release];

```

### 不要在dealloc中管理系统稀缺【scarce】资源
通常你不需要在dealloc中管理系统的稀缺资源，例如文件描述符、网络连接、缓冲区以及缓存等。实践中，你不应该假设dealloc一定会在预期的时机被调用。dealloc有可能会被延期调用，甚至不调用，也许因为bug，也许因为程序被终止了。
你应该有个类，它的实例来管理稀缺资源。你可以这样设计，当你不再需要这些资源的时，能告知这个实例“清理”【clean up】资源。然后你可以release这个实例，随后它的dealloc会被调用，但即使不调用，也不会再有问题了。
如果你尝试在dealloc处理这些资源，可能会引入问题，例如：
1. 依赖对象图销毁的顺序【Order dependencies on object graph tear-down】
对象图销毁的机制是无序的。你如果期望一个特定的顺序，会让程序变的脆弱。例如，一个对象被autorelease了，而不是release【 If an object is unexpectedly autoreleased rather than released for example】，销毁顺序可能会变化，从而引发不可预料的结果。
2. 稀缺资源不可复用。
内存泄露是bug，应该被修复，但通常不会立即造成致命后果。如果你的稀缺资源没有适时的被释放，可能会造成更严重的问题。例如，如果你的程序用光了文件描述符，用户的数据可能就无法保存了。
3. 清理资源可能在错误的线程上执行。
如果一个对象在非预期的时机被autorelease了，它可能会在相关的autorelease pool block所在的线程上销毁。如果一个资源要求必须在特定线程销毁，这种情况下可能会产生致命问题。

### 容器【Collections】会拥有其包含的对象
当你将一个对象添加到容器中时，容器会拥有这个对象。容器移除对象或容器自身要被释放时，它会放弃拥有对象。
例如，当你生成一个盛放数字的容器时，你可能有下面两种方式：
```
NSMutableArray *array = <#Get a mutable array#>;
NSUInteger i;
// ...
for (i = 0; i < 10; i++) {
    NSNumber *convenienceNumber = [NSNumber numberWithInteger:i];
    [array addObject:convenienceNumber];
}
```
这种情况下，没有使用alloc，你不需要release，你无需retain convenienceNumber，容器会去做。

```
NSMutableArray *array = <#Get a mutable array#>;
NSUInteger i;
// ...
for (i = 0; i < 10; i++) {
    NSNumber *allocedNumber = [[NSNumber alloc] initWithInteger:i];
    [array addObject:allocedNumber];
    [allocedNumber release];
}
```
这种情况下，你需要在for循环中release对象，容器在addObject:时会拥有对象，对象不会被销毁。
为了加深理解，你可以从设计容器的人的角度考虑，容器不想在稍后要使用对象时，发现对象已经被销毁了。因此在添加对象时retain，移除对象时release，容器dealloc时release对象，就不会有这样的问题。

### 拥有关系通过引用计数实现
拥有关系通过引用计数实现，每个对象都有引用计数。
- 当你创建对象时，它的引用计数置为1。
- 当你发送retain消息时，它的引用计数加1。
- 当你发送release消息时，它的引用计数减1。
- 当你发送autorelease消息时，它的引用计数会延期减1，时机是其所在的autorelease pool block结束的时候。
- 如果对象引用计数变为0，则会被销毁。

> 不要显式的查看对象的引用计数，结果可能有误导性。你不知道是不是系统对象也在拥有你感兴趣的对象。在调试内存管理问题时，你应该专注于让代码遵循内存管理的规则。










