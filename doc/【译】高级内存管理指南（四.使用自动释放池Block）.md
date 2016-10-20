[Advanced Memory Management Programming Guide 官档传送门](https://developer.apple.com/library/content/documentation/Cocoa/Conceptual/MemoryMgmt/Articles/MemoryMgmt.html#//apple_ref/doc/uid/10000011-SW1)

本文翻译自2012-07-17版的官档。
您可以在官档结尾的Document Revision History中查阅版本。
注：【】包含的部分引自原文。
## 使用自动释放池Block【Using Autorelease Pool Blocks】
自动释放池提供了一种机制，允许你延时release对象，这样你就可以解除拥有关系，但对象不会被立即销毁（例如从方法返回一个对象）。通常来讲，你不需要主动创建自动释放池，但在有的场景下，你必须自己创建，或者自己创建能带来额外的帮助。

### 关于自动释放池Block
自动释放池Block使用@autoreleasepool来标记，如下面例子所示。
```
@autoreleasepool {
    // Code that creates autoreleased objects.
}
```
在自动释放池Block结束的时候，之前在Block收到autorelease消息的对象，会收到release消息，每收到过一次autorelease，就会收到一次release【an object receives a release message for each time it was sent an autorelease message within the block】。
像其他的代码块一样，自动释放池Block也可以嵌套，如下：
```
@autoreleasepool {
    // . . .
    @autoreleasepool {
        // . . .
    }
    . . .
}
```
（你通常不会看到上面的代码块，一个源文件中自动释放池Block中的代码可能会调用另一个源文件中自动释放池Block中的代码）。对象在哪个自动释放池Block接到autorelease消息，就会在那个自动释放池Block结束时接到release消息。
Cocoa希望代码都要在自动释放池Block中执行，否则autorelease不会被释放，会引起内存泄露（如果你在自动释放池Block外发送了autorelease消息，Cocoa会记录一条错误日志）。AppKit和UIKit都在自动释放池Block处理事件循环（如鼠标点击或触摸）。你通常不需要自己创建自动释放池Block，甚至都看不到相关的创建代码。但在下面三种场景中，你可能要创建自己的自动释放池Block：
- 如果你的程序不基于UI framework，例如命令行工具。
- 如果你在循环中创建很多临时对象。
你可能需要在loop中创建一个自动释放池Block，在下一次迭代开始之前，释放本次迭代创建的对象。在loop中使用自动释放池Block可以降低程序内存占用。
- 如果你创建了次级线程。
你要在线程执行前创建自己的自动释放池Block，否则程序会有内存泄露（详细请参考后文的Autorelease Pool Blocks and Threads一节）。

### 使用局部自动释放池Block来降低内存占用
很多程序会创建自动释放的临时对象。这些对象增加了内存占用，一直持续到自动释放池Block结束。大多数情况下，临时对象累积到自动释放池Block结束不会增加太多开销。但有些时候，你可能创建大量的临时对象，大幅度的增加了内存占用，你想让他们更快的释放。这种情况下你可以创建自己的自动释放池Block。在block结束的时候，这些对象会被release，可能导致其被销毁，因此减少了程序的内存占用。
下面的例子展示了如何在for循环中使用局部自动释放池Block：
```
NSArray *urls = <# An array of file URLs #>;
for (NSURL *url in urls) {
 
    @autoreleasepool {
        NSError *error;
        NSString *fileContents = [NSString stringWithContentsOfURL:url
                                         encoding:NSUTF8StringEncoding error:&error];
        /* Process the string, creating and autoreleasing more objects. */
    }
}
```
for循环每次处理一个文件，每个在自动释放池Block中收到autorelease的对象（如fileContents），在block结束时都会收到release。
在自动释放池Block结束后，在block中autorelease的对象都应该被视为已释放的，不应再使用它或将它返回给方法的调用者。如果你必须在block外使用一个自动释放池Block中的临时对象，你应该在block中retain它，在block结束后再对其autorelease，如下面的例子所示：
```
– (id)findMatchingObject:(id)anObject {
 
    id match;
    while (match == nil) {
        @autoreleasepool {
 
            /* Do a search that creates a lot of temporary objects. */
            match = [self expensiveSearchForObject:anObject];
 
            if (match != nil) {
                [match retain]; /* Keep match around. */
            }
        }
    }
 
    return [match autorelease];   /* Let match go and return it. */
}
```
在block中给match对象发送retain，并在结束后发送autorelease，来扩展它的生命周期，并可以安全的作为方法的返回值。

### 自动释放池Block和线程
Cocoa中的每个线程维护自己的自动释放池Block栈。如果你的程序只基于Foundation【Foundation-only 】，或者你创建【detach】了线程，你需要自己创建自动释放池Block。
如果你的程序或线程会长期生存，并可能生成很多autorelease对象，你应该使用自动释放池Block（就像AppKit或UIKit在主线程中做的那样），否则autorelease对象会累积，增加程序的内存占用。如果你创建的线程没有Cocoa call（译注：没完全理解），那就不需要创建自动释放池Block。

> 注意：如果你创建的线程是基于POSIX API而非NSThread，你不能使用Cocoa（译注：没完全理解），除非Cocoa处于多线程模式。Cocoa在创建第一个NSThread后才会进入多线程模式。使用POSIX API之前，你的程序至少应该创建一个NSThread，这个NSThread对象可以执行后立即退出。你可以用NSThread的isMultiThreaded方法测试Cocoa是否处于多线程模式。



