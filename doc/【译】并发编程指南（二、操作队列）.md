[Concurrency Programming Guide 官档传送门](https://developer.apple.com/library/content/documentation/General/Conceptual/ConcurrencyProgrammingGuide/Introduction/Introduction.html#//apple_ref/doc/uid/TP40008091)

本文翻译自2012-12-13版的官档。
您可以在官档结尾的Document Revision History中查阅版本。
注：【】包含的部分引自原文。

## 操作队列【Operation Queues】
Cocoa的operation采用面向对象的方式来封装异步执行的任务。operation被设计为既可以和操作队列联合使用，也可以自己执行。因为基于Objective-C（后文简称OC），所以常用于OS X和iOS上基于Cocoa的应用。

本章展示了如何定义和使用operation。

### 关于operation
operation是Foundation库中NSOperation类的实例，你可以用它来封装任务。NSOperation类是个抽象类，你需要创建它的子类来完成具体工作。尽管它是抽象类，它还是提供了大量的基础代码，来减少子类的工作。除此之外，Foundation库还提供了两个NSOperation的子类，你可以直接使用它们来执行任务。表2-1列举了它们，并简要介绍了它们的用法。

表2-1 Foundation库的operation类
- NSInvocationOperation
NSInvocationOperation基于目标对象和相关selector生成实例，它适合已经有方法来执行任务的场景。由于不需要创建子类，这种方式可以动态地创建operation。
更多信息请参考后文“创建NSInvocationOperation对象”一节。
- NSBlockOperation
NSBlockOperation可以并发地执行单个或多个block，由于可以执行多个block，它采用了group的语义，只要所有block都完成，operation才算完成。
更多信息请参考后文“创建NSBlockOperation对象”一节。
- NSOperation
NSOperation是自定义operation的基类。生成其子类，你可以完全控制operation的实现，包括修改其执行任务或报告状态的默认实现。
更多信息请参考后文的“自定义operation”一节。

所有的operation都支持下述的关键功能：
- 支持operation间建立依赖关系，被依赖的operation完成后，依赖其的operation才开始执行。更多信息请参考后文的“配置operation的依赖关系”一节。
- 支持可选的完成block，该block会在operation的主任务完成后调用。更多信息请参考后文的“设置完成block”一节。
- 支持KVO监视任务的执行状态。关于KVO，请参考[Key-Value Observing Programming Guide](https://developer.apple.com/library/content/documentation/Cocoa/Conceptual/KeyValueObserving/KeyValueObserving.html#//apple_ref/doc/uid/10000177i)文档。
- 支持operation的优先级，优先级可以影响operation的执行顺序。更多信息请参考后文的“改变operation执行优先级”一节。
- 支持取消任务，如何取消任务，请参考后文“取消任务”一节，自定义的operation如何支持取消任务，请参考后文“响应取消事件”一节。

operation能帮你提升并发操作的级别（译注：粒度），operation也是组织应用行为的好方式，你可以使用operation把行为封装为独立的任务。你还可以将代码从主线程剥离出来，作为任务异步地在后台线程执行。

### 并发与非并发的operation
通常operation是放入操作队列中异步执行，但这不是绝对的。你可以直接调用operation的start方法来执行它，但不能保证operation是异步执的。NSOperation会根据isConcurrent方法来决定是否相对于【with respect to】start被调用的线程异步执行。该方法默认返回NO，也就意味着operation会在start调用线程上同步执行。

如果想要实现并发operation（也就是说，你的operation相对于被调用的线程异步执行），那么你需要写额外的代码来异步执行operation。例如，你可能需要生成额外的线程，或是调用系统的异步函数，或使用其他方式来保证operation的start方法不必等任务结束就返回。

通常情况下你不需要实现并发operation。如果你将operation提交给操作队列，就不需要实现。及时你提交的是非并发operation，队列也会用子线程去执行它。也就是操作队列会保证你提交的任务是异步执行的。只有在你不将operation提交给操作队列这种极端情况下，你才需要自己实现并发operation。

更多关于实现并发operation的信息，请参考后文的“配置并发operation”一节。以及[NSOperation Class Reference](https://developer.apple.com/reference/foundation/operation)文档。

### 创建NSInvocationOperation对象
NSInvocationOperation是NSOperation子类。它在执行的时候，会调用你指定对象的制定selector。使用此类可避免生成大量的自定义operation类。尤其是当你修改应用，并且该应用已经拥有执行任务的对象和方法的场景。还有一种适合的场景是，你根据环境动态的选择要执行的方法。例如：你需要根据用户输入动态选择要执行的selector。

生成NSInvocationOperation对象的过程非常简单，创建它的实例，并在初始化方法中传入目标对象和目标selector即可。代码清单2-1中，展示了如何生成NSInvocationOperation对象，taskWithData:方法创建了对象，并提供了目标对象和目标方法，目标方法包含任务的业务逻辑。

代码清单2-1 生成NSInvocationOperation对象
```
@implementation MyCustomClass
- (NSOperation*)taskWithData:(id)data {
    NSInvocationOperation* theOp = [[NSInvocationOperation alloc] initWithTarget:self
                    selector:@selector(myTaskMethod:) object:data];
 
   return theOp;
}
 
// This is the method that does the actual work of the task.
- (void)myTaskMethod:(id)data {
    // Perform the task.
}
@end

```

### 生成NSBlockOperation对象
NSBlockOperation是NSOperation的子类，它包装了单个或多个block。如果你的应用在使用操作队列，你想使用block，又不想引入GCD的话，可以用此类将block包装为面向对象的operation。或者你想使用block，又想理由operation和操作队列的一些特性，例如任务间的依赖、KVO通知等功能，这种场景下，也可以选择使用此类。

当你创建NSBlockOperation时，你至少要传入一个block，你可以添加多个block。当operation被执行时，它会将所有block提交到一个默认级别的并发派发队列中，当所有block执行完成后，operation才会将自己标记为完成。因此，你可以用此类来追踪一组block的执行，很类似线程通过join其他线程来合并执行结果。区别在于NSBlockOperation本身也运行在子线程上，等待其完成的期间，其他线程可以继续工作。

代码清单2-2展示了如何生成NSBlockOperation对象。这个block没有参数也没有返回结果。

代码清单2-2 生成NSBlockOperation对象
```
NSBlockOperation* theOp = [NSBlockOperation blockOperationWithBlock: ^{
      NSLog(@"Beginning operation.\n");
      // Do some work.
   }];
```
生成NSBlockOperation对象后，你可以使用addExecutionBlock:方法继续添加block。如果你想要block串行执行，你需要直接将它们添加到相关的派发队列。

### 自定义operation
如果NSInvocationOperation和NSBlockOperation不能满足你的需求，你可以创建NSOperation的子类来实现你的需求。NSOperation提供了很多子类的重写点，也提供了很多基础代码（译注：非业务代码），例如operation间的依赖，KVO通知等。但有些情况下，你也需要自己实现一些基础代码来满足特定的需求，实现额外基础代码的多少，通常取决于你是否要创建并发operation。

定义非并发operation要比并发operation简单的多。对于非并发operation来说，你所要做的就是执行主任务和响应取消事件，NSOperation提供的基础代码会为你处理其他的工作。而对于并发operation来说，你需要替换掉一些现有的基础代码。接下来的章节会为你展示两种operation的实现。

#### 执行主任务【Performing the Main Task】
每个operation至少要实现以下两个方法：
- 自定义的init系方法（译注：例如initWithData:）
- main方法

你使用自定义的init系方法来初始化任务，使用main方法来执行任务。你可以根据需要实现额外的方法，例如：
- main方法中要调用的自定义方法
- property的访问方法
- NSCoding协议方法，来让operation可以序列化

代码清单2-3中展示了NSOperation的自定义子类（这份代码只提供实现子类常见的方法实现，没有涉及取消任务，取消任务请参考后文“响应取消事件”一节）。初始化方法获取了data参数，并用成员变量持有它。main方法会使用data执行具体任务。

代码2-3 自定义简单的NSOperation子类
```
@interface MyNonConcurrentOperation : NSOperation
@property id (strong) myData;
-(id)initWithData:(id)data;
@end
 
@implementation MyNonConcurrentOperation
- (id)initWithData:(id)data {
   if (self = [super init])
      myData = data;
   return self;
}
 
-(void)main {
   @try {
      // Do some work on myData and report the results.
   }
   @catch(...) {
      // Do not rethrow exceptions.
   }
}
@end
```
更多实现细节请参考[NSOperationSample](https://developer.apple.com/library/content/samplecode/NSOperationSample/Introduction/Intro.html#//apple_ref/doc/uid/DTS10004184)官方示例代码。

#### 响应取消事件
operation开始执行后，它会持续执行直到结束，或者被显式的取消。取消事件可能发生在任意时间，甚至是operation开始执行之前。尽管NSOperation提供了取消的方式，但是必要时还是要主动检查取消事件。如果operation彻底终结了，就没有办法再去回收operation中创建的资源。因此，operation最好能检查到取消事件，然后能在执行中途优雅的退出。

让operation支持取消，唯一要做的就是在任务执行过程中适时的去检查isCancelled方法的值，只要其返回YES，就退出任务。不管你创建NSOperation的子类，还是使用Foundation提供的子类都是一样。isCancelled是个非常轻量级的方法，因此你可以频繁的调用，不用担心性能。设计operation类时，你可以考虑在下列情况下检查isCancelled方法：
- 开始执行任务之前
- 循环的每次迭代之前，如果每次迭代都执行长期操作，那么可以更频繁的检查isCancelled
- 每个比较容易中断任务的点【easy to abort the operation】

代码清单2-4展示了如何在main方法中响应取消事件。示例中，while的每个迭代都检查isCancelled，因此可以让operation在工作开始执行前或进一步执行前退出。

代码清单2-4 响应取消事件
```
- (void)main {
   @try {
      BOOL isDone = NO;
 
      while (![self isCancelled] && !isDone) {
          // Do some work and set isDone to YES when finished
      }
   }
   @catch(...) {
      // Do not rethrow exceptions.
   }
}
Although the preceding example contains no cleanup code, your own code should be sure to free up any resources that were allocated by your custom code.
```
上面的示例中并没有清理资源的代码，但你的代码中要确保释放了执行过程中创建的资源。

#### 配置并发operation
operation默认是同步执行的，他们在调用start方法的线程上同步的执行任务。由于操作队列会在子线程上调用operation的start方法，因此提交给操作队列的operation会异步执行。但如果你想不依赖操作队列手动执行operation，并且希望是异步执行，你需要进行适当的处理来确保你的operation是并发operation。
表2-2列举了生成并发operation需要重写的方法

表2-2 并发operation需要重写的方法
- start
（必须）所有并发operation必须重写这个方法，使用自定义代码替换掉默认实现。你通过调用start来手动执行operation。因此这个方法是自定义代码的起点，你需要在这个方法中建立线程或其他异步执行环境。你的实现**不能**调用super的实现。
- main
（可选）这个方法通常用于实现任务的具体逻辑。虽然你也可以在start方法中实现业务代码。但在main方法中实现可以让建立环境和执行业务分离的更为清晰。
- isExecuting/isFinished
（必须）并发operation有责任建立异步执行环境，以及向外面报告自己的执行状态。因此并发operation必须管理状态信息来了解什么时候任务正在执行，什么时候执行结束。它需要用这些方法报告状态。
你必须保证这些方法是线程安全（译注：多线程可以同步访问）的，你也需要生成相关的KVO通知来报告状态。
- isConcurrent
（必须）指示是否是并发operation，重写这个方法并返回YES。

这一节剩下的部分展示了MyOperation子类的实现。它展示了实现并发operation需要的最基本的代码。MyOperation简单地在自己创建的线程上执行main方法。main方法中实现具体工作，但具体工作是什么在本例中并不重要。本例重点展示实现并发operation的基础代码。
代码清单2-5展示了MyOperation类的部分实现。isConcurrent、 isExecuting和isFinished方法的实现都非常简单。 isFinished只是简单返回了YES，其他两个方法直接返回了相应的实例变量的值。

代码清单2-5 定义并发operation
```
@interface MyOperation : NSOperation {
    BOOL        executing;
    BOOL        finished;
}
- (void)completeOperation;
@end
 
@implementation MyOperation
- (id)init {
    self = [super init];
    if (self) {
        executing = NO;
        finished = NO;
    }
    return self;
}
 
- (BOOL)isConcurrent {
    return YES;
}
 
- (BOOL)isExecuting {
    return executing;
}
 
- (BOOL)isFinished {
    return finished;
}
@end
```

代码清单2-6展示了start方法的实现。start中只写了必须的代码。本例中，start简单的启动了一个新线程，并在该线程上调用main方法。并在适当的点加入了isFinished和isExecuting的KVO通知。完成这些工作，start方法就返回了，具体工作由新启动的线程执行。

代码清单2-6 start方法
```
- (void)start {
   // Always check for cancellation before launching the task.
   if ([self isCancelled])
   {
      // Must move the operation to the finished state if it is canceled.
      [self willChangeValueForKey:@"isFinished"];
      finished = YES;
      [self didChangeValueForKey:@"isFinished"];
      return;
   }
 
   // If the operation is not canceled, begin executing the task.
   [self willChangeValueForKey:@"isExecuting"];
   [NSThread detachNewThreadSelector:@selector(main) toTarget:self withObject:nil];
   executing = YES;
   [self didChangeValueForKey:@"isExecuting"];
}
```

代码清单2-7展示了MyOperation的其余部分的实现。在代码清单2-6中我们了解到，main方法是新线程的执行起点。它包含了实际业务逻辑，并且在执行完成后调用自定义的completeOperation。completeOperation方法中生成isExecuting和isFinishedKVO需要的通知。

代码清单2-7 任务完成时更新operation
```
- (void)main {
   @try {
 
       // Do the main work of the operation here.
 
       [self completeOperation];
   }
   @catch(...) {
      // Do not rethrow exceptions.
   }
}
 
- (void)completeOperation {
    [self willChangeValueForKey:@"isFinished"];
    [self willChangeValueForKey:@"isExecuting"];
 
    executing = NO;
    finished = YES;
 
    [self didChangeValueForKey:@"isExecuting"];
    [self didChangeValueForKey:@"isFinished"];
}
```
即使operation被取消，你也需要通知KVO观察者工作完成。其他operation可能依赖你的operation，这种情况下它会监视你的isFinished。只有当收到finish通知时，依赖你的operation来开始执行。因此，如果没有实现KVO通知，可能会造成其他operation无法执行。

#### 维护KVO的约定
NSOperation的下述key path遵循KVO的约定
- isCancelled
- isConcurrent
- isExecuting
- isFinished
- isReady
- dependencies
- queuePriority
- completionBlock

如果你重写start方法，或有重写main方法之外的其他定制，你要确保上述的key path都遵循KVO约定。重写start方法时，主要关注isExecuting和isFinished。这两个key path是重写这个方法时最容易受影响的。

如果你要重写依赖相关的方法。你可以重写isReady，并强制它返回NO，直到满足了依赖条件才返回YES。（如果你实现了自定义的依赖，但还想要利用默认的依赖机制，那么你重写isReady方法时还要调用super）。当准备状态改变时，会触发isReady的KVO通知。除非你重写了addDependency:和removeDependency:，否则你不用关心dependencies的KVO通知。 

虽然你也可以生产其他key path的KVO通知，但一般不需要考虑。如果你需要取消operation，你简单地调用cancel方法即可。类似地，你一般也不需要修改operation的队列优先级信息。最后，除非你确实要动态改变并发状态，否则你不用关心isConcurrent key path。

更多关于KVO的信息，请参考[Key-Value Observing Programming Guide](https://developer.apple.com/library/content/documentation/Cocoa/Conceptual/KeyValueObserving/KeyValueObserving.html#//apple_ref/doc/uid/10000177i)文档。

### 自定义operation的执行行为
对operation的配置发生在创建之后，将其加入队列之前。本节介绍的配置对所有的operation都使用，不管是NSOperation的自定义子类，还是Foundation提供的子类。

#### 配置依赖关系
依赖是一种将operation串行执行的方式。operation会在其依赖的operation完成之后才开始执行。因此，你可以构建简单的1对1的依赖，也可以构造更复杂的依赖图。

使用NSOperation的addDependency:方法来建立依赖关系。这个方法会在两个operation间建立单向的依赖关系。依赖意味着operation要等其依赖的operation完成之后才开始执行。拥有依赖关系的两个operation不需要在同一个操作队列中。operation会管理自己的依赖关系，因此你可以创建operation及其依赖，并将operation提交到不同的操作队列中。唯一的问题是，你不应该生成循环依赖，这是编程级的错误，要在运行前就处理掉。

当一个operation所依赖的operation全部完成后，这个operation就进入了就绪态（如果你自定义了isReady方法中的行为，就绪态信息由你的逻辑决定）。这种状态下，如果你的operation在队列中，队列可能在任意时刻启动它，如果你打算手动执行，此时你可以调用operation的start方法。

> 重要：你应该在operation执行或添加到操作队列之前为其添加依赖，之后添加的依赖无法阻止其执行。

依赖机制是依靠operation状态的KVO通知。如果你自定义了operation的行为，你要确保实现了恰当的KVO通知，以避免影响依赖机制。更多operation KVO的信息，请参考前文的“维护KVO约定”一节。更多配置依赖的信息，请参考[NSOperation Class Reference](https://developer.apple.com/reference/foundation/operation)。

#### 改变operation的执行优先级
对加入操作队列的operation来说，执行顺序首先取决于是否准备就绪，其次取决于优先级。是否就绪取决于operation间的依赖关系，而优先级是operation自己的属性。默认的话，所有operation都拥有"normal"优先级，你可以使用setQueuePriority:方法调整优先级。

优先级只针对同一个操作队列。如果你有多个操作队列，每个队列中operation的优先级是独立的。因此，某个队列中低优先级的operation是可能先于另一个队列中高优先级的operation执行的。

优先级不是依赖的替代品。只有operation进入就绪态，优先级才起作用。例如，队列中有两个不同优先级的operation都进入了就绪态，那么高优先级的先执行。然而，如果队列中高优先级的operation还没准备好，低优先级的operation已经进入了就绪态，那么执行低优先级的operation。如果你想确保一个operation在另一个operation完成之后执行，可以考虑使用依赖。更多信息可参考上文的“配置依赖关系”一节。

#### 改变底层线程优先级
OS X v10.6及以后，可以配置operation底层执行线程的优先级。线程策调度策略是由内核管理的，不过通常高优先级线程会有更多的机会执行。你可以指定operation的线程优先级，优先级的值在0.0到1.0之间，0.0代表最低，1.0代表最高。如果你没有明确指定线程优先级，默认是0.5。

你可以调用operation的setThreadPriority:方法来设置线程优先级，注意要在加入操作队列之前或手动执行之前。当operation被执行时。start的默认实现会使用该值设置底层的线程。该优先级只在main方法执行的过程中有效，其他代码均按默认优先级（包括结束时的回调block）。如果你实现并发operation，重写start方法时要注意手动配置该优先级。

#### 设置完成block
OS X v10.6及以后，operation可以在主任务执行完毕后，执行一个完成block。你可以在完成block中执行任何不想加入到主任务中的代码。例如，你可能使用完成block来通知观察者任务已完成。并发operation可以使用block生成相关的KVO通知。

使用NSOperation的setCompletionBlock:方法来设置完成block，block不能有参数和返回值。

### 实现operation的小提示
尽管operation很容易实现，但还是有些注意点。接下来的几节描述了你实现operation时要注意的一些点。

#### operation中的内存管理
接下来的小节描述了operation中管理内存的核心因素。更多信息请参考[Advanced Memory Management Programming Guide](https://developer.apple.com/library/content/documentation/Cocoa/Conceptual/MemoryMgmt/Articles/MemoryMgmt.html#//apple_ref/doc/uid/10000011i)文档。

##### 避免使用线程存储域【Per-Thread Storage】
尽管多数operation在子线程上执行，对多数非并发operation来说，线程都是操作队列提供的。如果队列为你提供了线程，你应该认为队列拥有线程，而不是你的operation。你不应该将数据等和不是由你创建的线程绑定。队列提供或收回线程来满足系统和应用的需求。因此，使用线程存储域传递数据是不可靠的。

operation任何情况下都不应该使用线程存储域。当你初始化operation时，你应该给它提供了执行工作所需要的一切事物。因此，operation应该拥有自己的数据存储处，来存储输入或生成的数据，直到数据返回调用方，或数据不再有用。

##### 你应该持有operation的引用
operation通常是异步执行的，你不应该创建完毕就忘记它，它们依然是属于你管理的对象。如果operation结束时还要使用结果数据，那么管理它就更为必要。

你来管理operation的原因是你可能无法从操作队列中获取它的引用。队列会尽早地执行加入其中的任务，很多时候，operation一加入队列就开始执行了，当你的代码想从操作队列中获取operation的引用时，operation可能已经执行完毕并从队列中移除了。

#### 处理错误和异常
由于operation本质上是你的应用所创建的独立实体，你有责任处理其可能出现的错误和异常。在OS X v10.6及以后，默认的start实现不会再捕获异常（之前的版本，start方法会捕获并抛弃【suppress】异常），你的代码需要自己捕获及处理异常，以及坚持错误并通知应用的其他部分。如果你重写了start方法，你也需要在实现中捕获处理异常，避免其进入到底层线程的域中。

你需要处理几种错误场景如下：
- 检查Unix errno-style错误码
- 检查函数或方法显式返回的错误码
- 捕获你的代码或系统库抛出的异常
- 捕获NSOperation本身抛出的异常，如下：
    - operation没有进入就绪态，start方法就被调用
    - operation已经执行或完成，start方法又被调用
    - operation已经执行，向其添加完成block    
    - 尝试从被取消的NSInvocationOperation对象中获取结果
    
如果你的自定义代码遇到了异常或错误，你应该进行处理，避免其传播到程序的其他部分。NSOperation没有提供显式的方法来让你把异常等传给程序的其他部分。因此，如果这些信息有用，你应该提供必要的代码。

### 确定operation适用的领域
尽管你可以向操作队列添加大量的operation，但这么做是不使用的。和其他对象一样，operation也要消耗内存，执行时也有开销。如果每个operation只做了很少量的工作，你创建了巨量的这样的operation，你可能发现执行任务比实际所需消耗了更多的时间。如果你的应用已经使用了很多内存【already memory-constrained】，你可能发现内存中巨量的operation降低了性能。

有效地使用operation的要点是找到operation数量和计算机忙碌度的平衡。确保你的operation数量是合理的。例如，你的应用生成100个operation去处理100个不同的值，可以考虑生成10个operation，每个operation处理10个值。

你应该避免同时向队列加入大量任务，或快速地连续向队列加入任务，队列的处理速度可能跟不上。你可以采用批量加入任务的方式。当一批任务执行完毕后，使用完成block告知你的应用生成新的一批任务。如果你有很多工作，你当然希望队列始终是满的，计算机始终是忙碌的，但你不会想同时创建了大量的任务以至于内存耗尽。

当然，你创建的operation数量，每个operation中工作的多少， 还是取决于你的程序需求。你应该经常用工具（如Instruments）来帮助自己找到效率和速度间的平衡。更多关于Instruments和性能的信息，可以参考[Performance Overview](https://developer.apple.com/library/content/documentation/Performance/Conceptual/PerformanceOverview/Introduction/Introduction.html#//apple_ref/doc/uid/TP40001410)文档。

### 执行operation
总起来说，你的应用还是需要operation来完成工作。这一节，你将学会使用几种方式执行operation，并且在运行期管理它们。

#### 将operation加入操作队列
目前，最简单的方式是使用操作队列执行operation。操作队列是NSOperationQueue的实例。你的应用有责任创建和管理所需的队列。理论上应用可以使用任意数量的队伍，但实际上同时执行的operation数量是有限制的。操作队列和系统根据CPU核心和系统负载来控制同时执行的operation数量。因此，创建额外的队列未必能执行更多的operation。

创建操作队列和创建其他对象没什么区别，如下：
```
NSOperationQueue* aQueue = [[NSOperationQueue alloc] init];
```

使用操作队列的addOperation:方法将operation加入队列。OS X v10.6及以后，你可以使用addOperations:waitUntilFinished:方法加入一组operation，或者你可以直接使用addOperationWithBlock:方法将block加入队列。这些方法都会把operation或一组operation加入队列，并通知队列执行。多数情况下，operation加入队列不久后就开始执行，不过有些时候，队列会延迟operation的执行。例如operation依赖其他operation，但其他operation没有执行完。或者执行队列本身被挂起，或者达到最大并发数。下面的代码展示了向队列添加operation的基本语法。
```
[aQueue addOperation:anOp]; // Add a single operation
[aQueue addOperations:anArrayOfOps waitUntilFinished:NO]; // Add multiple operations
[aQueue addOperationWithBlock:^{
   /* Do something. */
}];
```
> 重要：一旦将operation加入队列，就不要再修改它。加入队列后，operation随时都可能开始执行，此时修改它的依赖关系和数据可能造成不良的后果。如果你想知道operation的状态，你可以使用NSOperation的方法来检查operation是运行中、等待运行还是已经完成了。

尽管NSOperationQueue被设计为并发队列，但你可以强制让任务依次执行。setMaxConcurrentOperationCount:方法用于配置最大并发数，如果你设置最大并发数为1的话，那么同一时间只能有一个operation执行。尽管同一时间只能执行一个operation，但operation执行顺序并不能保证，还依赖其他因素。比如operation是否进入就绪态以及优先级。因此，依次执行的并行队列无法真正达到GCD中串行派发队列的效果。如果你的任务执行顺序非常重要，你需要使用operation依赖来建立恰当的执行顺序。具体可参考前文的“配置依赖关系”一节。

更多操作队列的信息，请参考[NSOperationQueue Class Reference](https://developer.apple.com/reference/foundation/nsoperationqueue)文档。更多串行派发队列的信息，请参考[Creating Serial Dispatch Queues](https://developer.apple.com/library/content/documentation/General/Conceptual/ConcurrencyProgrammingGuide/OperationQueues/OperationQueues.html#//apple_ref/doc/uid/TP40008091-CH102-SW6)一章。

#### 手动执行operation
尽管操作队列是最方便的执行operation的方式，有时候可能还是会手动执行operation。如果你选择了手动执行，需要多加注意。详细地说，operation必须进入就绪态，你必须调用它的start的方法。

operation的isReady方法返回YES时，才被认为准备就绪了。iSReady方法会集成到依赖机制中，并提供依赖的状态，只有operation的依赖全部完成，它才可以执行。

如果你选择手动执行operation，你应该调用它的start方法，而不是main或其他方法。因为start方法会在执行自定义方法前做一些安全检查。详细地说，默认的start方法会生成operation处理依赖所需的KVO通知。这个方法也会避免在operation被取消后再尝试执行任务，并且在operation处于非就绪态时抛出异常。

如果程序定义了并发operation，你还应该在启动它们之前【prior to launching them】调用isConcurrent方法。如果这个方法返回NO，你的代码要决定是在当前线程同步的执行任务，还是先创建子线程。当然，实现什么样的检查还是取决于你。

代码清单2-8展示了手动执行operation前的简单检查。如果方法返回了NO，就注册timer稍后再次检查。你可以反复注册timer直到方法返回YES，operation被取消可以产生这一结果。

代码清单2-8 手动执行operation
```
- (BOOL)performOperation:(NSOperation*)anOp
{
   BOOL        ranIt = NO;
 
   if ([anOp isReady] && ![anOp isCancelled])
   {
      if (![anOp isConcurrent])
         [anOp start];
      else
         [NSThread detachNewThreadSelector:@selector(start)
                   toTarget:anOp withObject:nil];
      ranIt = YES;
   }
   else if ([anOp isCancelled])
   {
      // If it was canceled before it was started,
      //  move the operation to the finished state.
      [self willChangeValueForKey:@"isFinished"];
      [self willChangeValueForKey:@"isExecuting"];
      executing = NO;
      finished = YES;
      [self didChangeValueForKey:@"isExecuting"];
      [self didChangeValueForKey:@"isFinished"];
 
      // Set ranIt to YES to prevent the operation from
      // being passed to this method again in the future.
      ranIt = YES;
   }
   return ranIt;
}
```

#### 取消operation
operation一旦加入队列，就被队列拥有，不能被移除了。唯一从队列中移除它的方式就是取消它。你可以使用cancel方法取消单个的operation，或用队列的cancelAllOperations方法取消队列中所有的operation。

你只应该在确认不需要operation的时候才取消它。取消一个operation会将其标记为取消态，它将无法再执行。由于取消态也被认为是完成态，因此依赖它的其他operation接到KVO通知后，会清除对它的依赖。因此，通常情况下，我们是在接到某些事件（例如程序将推出或用户显式要求取消）后取消所有任务，而不是取消一个特定的任务。

#### 等待operation完成
为了更好的性能，operation一般是异步执行，从而让你程序可以同时处理其他工作。但如果你的程序需要等待operation处理的结果，你可以用NSOperation的waitUntilFinished方法阻塞当前线程，直到operation完成。通常，你不应该使用这种方式。阻塞线程可能是个方便的方式，但它引入了串行，会限制并发执行operation的数量。
> 重要：你永远不应该在主线程等待operation执行。你只能在次级线程或其他operation中这样做。阻塞主线程会让你的程序无法想用用户操作。

除了等待单个operation执行完成，你还可以使用队列的waitUntilAllOperationsAreFinished 方法来等待所有的operation完成。在你等待所有operation完成时，其他线程又会想队列添加operation，你的等待时间会随之延长。

#### 挂起和回复队列
如果你想临时挂起所有operation，你可以使用队列的setSuspended:方法直接挂起队列。挂起队列不会导致正在执行的任务中途停止，它会阻止新的任务启动。你可以在用户要求的时候挂起队列暂停正在进行的工作，用户稍后可能会要求恢复这项工作。

 