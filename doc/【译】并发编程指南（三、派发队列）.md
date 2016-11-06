[Concurrency Programming Guide 官档传送门](https://developer.apple.com/library/content/documentation/General/Conceptual/ConcurrencyProgrammingGuide/Introduction/Introduction.html#//apple_ref/doc/uid/TP40008091)

本文翻译自2012-12-13版的官档。
您可以在官档结尾的Document Revision History中查阅版本。
注：【】包含的部分引自原文。

## 派发队列【Dispatch Queues】
GCD（Grand Central Dispatch）的派发队列是执行任务的优秀工具。派发队列可以按调用方需求灵活地同步或异步执行block。可以说，绝大部分需要再次级线程执行的任务，你都可以用派发队列处理。派发队列不仅易于使用，而且比直接使用线程代码更高效。

本章介绍了派发队列的知识，以及如何使用它来执行通用任务。如果你想从直接使用线程的代码迁移到GCD，请参考[Migrating Away from Threads](https://developer.apple.com/library/content/documentation/General/Conceptual/ConcurrencyProgrammingGuide/ThreadMigration/ThreadMigration.html#//apple_ref/doc/uid/TP40008091-CH105-SW1)一章。

### 关于派发队列
- 对于执行异步并发的任务，派发队列是个简单的方式。所谓任务就是你的程序要执行的部分工作。例如，定义了一个计算任务，创建或修改一个数据机构，处理从文件中读取的数据等等。你可以将工作代码放在函数或block中来定义一个任务，然后将其加入派发队列。

派发队列是个类似于对象的数据结构，它可以管理你提交的任务。所有的派发队列都是先进先出的数据结构。因此，任务的执行顺序和你添加任务的顺序是一致的。GCD已经为了提供了几个队列，当然你可以可以按需创建自己的队列。表3-1列举了派发队列的类型，以及如何去使用它们。

表3-1 派发队列的类型
- 串行【Serial】
串行队列（也被认为是私有队列【also known as private dispatch queues】，译注：不是很理解）按照任务添加的顺序，依次执行任务，同一时间只执行一个任务。当前任务在派发队列提供的特定线程上执行（不同任务可能执行在不同线程上）。串行队列通常用于同步访问特定资源。
你可以创建很多串行队列，每个队列执行任务相对于其他队列都是并发的。例如，你创建了4个串行队列，每个队列同一时间只能执行一个任务，但你依然可以并发的执行4个任务，将这4个任务派发到不同队列即可。更多信息请参考后文“创建串行队列”一节。
- 并行【Concurrent】
并行队列（也被认为是全局队列【also known as a type of global dispatch queue】）可以并发地执行多个任务。不过任务的启动顺序呢还是和添加的顺序一致。当前执行的任务运行在队列提供的特定线程上。特定时刻执行的任务数量要希来系统条件。
iOS5及以后，你可以使用参数DISPATCH_QUEUE_CONCURRENT创建自己的并发队列（译注：根据此句可推测，之前并发队列应该都是系统全局的，自己只能创建串行队列，因此两种队列有前文中的其他叫法）。除此之外，系统还提供了4个预定义的全局并发队列。更多信息请参考后文“获取全局并发队列”一节。
- 主队列【Main dispatch queue】
主队列是个全局串行队列，它的任务会在应用的主线程执行。主队列和的应用的Runloop合作，将任务穿插在Runloop要执行的其他任务间执行。因此任务将运行在主线程上，主队列通常作为程序的一个关键同步点【key synchronization point 】。
尽管你不需要创建主队列，但你还是要确保程序正确的清理它【drains it appropriately】。更多信息请参考后文的“在主线程上执行任务”一节。

对于引入并发而言，派发队列比直接使用线程有很多优势。优势之一就提供了简单易用的工作队列编程模型【work-queue programming model】。直接使用线程的话，你除了要写业务代码之外，还要写创建、管理线程的代码。派发队列可以让你将注意力放在业务代码上，而不用关心线程的管理等工作，系统会为你处理这些事。另一个优势就是系统比应用更擅长管理线程，系统可以根据当前资源和系统条件动态地的调节线程数。另外，相比你自己创建线程来说，系统能更快的启动你的任务。

你可以认为使用派发队列来重写代码会很困难，其实它的代码比直接使用线程的代码简单的多。使用派发队列的关键就是封装自包含【self-contained】的可以异步执行的任务（其实这对派发队列和使用线程来讲，都是关键）。派发队列还有个优势，是行为更可预期。如果你使用线程来执行任务，两个任务访问一个共享资源，你要使用锁来保证两个任务不会同时修改这个共享资源。但使用派发队列的话，你将这两个任务加入一个串行队列就可以了。并且，使用串行队列比锁更加高效，因为不管是否抢占式调度【both the contested and uncontested cases】，锁都会导致一个昂贵的内核trap，而队列工作在你的程序的进程控件，它只在绝对必需的情况下，才会访问内核【calls down to the kernel】。

你可能会想，两个任务放入串行队列，就不是并发了，这么想是对的，但你要考虑到，两个线程使用锁时，其并发性也消失了，或至少显著减少了。更为重要的是，使用线程会创建两个线程，内核空间和用户空间的内存都会被占用（译注：操作系统知识不够，不理解）。派发队列不用为其使用的线程提供内存，并且它使用的线程会保持忙绿，不会阻塞（译注：不太理解，推测是这些完全由系统控制，你的进程不用时，会给别的进程使用，所以可以保持忙碌）。

使用派发队列，还有一些要注意的点，如下：
- 派发队列执行任务时，相对于其他队列来说，是并发的，串行的概念只是针对同一个队列中的任务。
- 系统决定了同一时间执行任务的数量。因此，将100个任务派发到100队列中，不一定能保证它们同时并发执行（除非CPU有100个或更多的可用的核心）。
- 系统决定执行哪个任务时，会参考队列的优先级。更多信息请参考后文“为队列提供清理函数”一节。
- 任务加入队列时必须已经准备好被执行（如果你使用过Cocoa的操作队列，你会发现这和操作队列的策略不一样）。
- 私有派发队列【Private dispatch queues】（译注：个人理解为非系统提供的，自己创建的队列）是引用计数对象。要注意在代码中持有队列，也要注意派发源【dispatch sources】注册到队列上时，也会增加其计数。因此你在管理队列时，要注意是否所有派发源都取消了，以及是否对队列的retain都有对应的release。（译注：iOS6及以后，ARC接管了派发对象的内存管理）。更多队列内存管理的信息，请参考后文的“派发队列的内存管理”一节。更多关于派发源的信息，请参考[About Dispatch Sources](https://developer.apple.com/library/content/documentation/General/Conceptual/ConcurrencyProgrammingGuide/GCDWorkQueues/GCDWorkQueues.html#//apple_ref/doc/uid/TP40008091-CH103-SW12)一章。

更多派发队列接口的信息，请参考Grand Central Dispatch (GCD) Reference文档(译注：翻译时，该链接已不可用)。

### 队列相关技术
GCD为派发队列提供了一些技术，来帮你管理代码。表3-2列举了这些技术。

表3-2 派发队列相关技术
- 派发组【Dispatch groups】
派发组可以用于监视一组block的完成（你可以按需监视同步或异步的block）。有的任务需要依赖其他一组任务完成，派发组为这种场景提供了一种有效的同步机制。更多信息，请参考后文“等待一组任务完成”一节。
- 派发信号量【Dispatch semaphores】
派发信号量类似于传统的信号量，不过它更高效。派发信号量机制，只会在线程因为信号量不存在而阻塞时，才会访问内核。如果信号量存在，不会访问内核。更多信息，请参考后文“使用派发信号量调节有限资源”一节。
- 派发源【Dispatch sources】
派发源为了响应系统事件，会生成相关的通知。你可以使用派发源来监视系统事件，例如进程通知、信号、描述符事件等等。事件发生时，派发源会将相关任务提交到制定的派发队列处理。更多信息请参考[Dispatch Sources](https://developer.apple.com/library/content/documentation/General/Conceptual/ConcurrencyProgrammingGuide/GCDWorkQueues/GCDWorkQueues.html#//apple_ref/doc/uid/TP40008091-CH103-SW1)一章。

### 使用block实现任务
block是C衍生语言中的功能，你可以在C、C++、Objective-C（后文简称OC）中使用。block让定义自包含的任务变得更加轻松。尽管看上去像是函数指针，block的底层实现其实更类似于对象，并被编译器管理。编译器会将block的代码和数据封装为一个可以在堆上存储并可以在程序中传递的形式。

block有一个重要的优势，它可以访问自己域外的变量。如果你在函数或方法中定义了block，block类似于在其中定义的传统代码块（译注：推测为{}包含的代码块）。例如，block可以访问父域中定义的变量。block中访问的变量会被复制到堆上的block结构体中，以便后续访问。block被加入派发队列后，这些值一般是只读的。但同步执行block，也可以通过父域中使用__block修饰的变量，向父域返回数据。

你可以像定义函数指针一样定义内联的block，两者唯一的不同是函数指针使用*标识，而block使用^标识。和函数指针一样，你可以向block传参数，也可以让block返回值。 代码清单3-1展示如何定义block，以及如何同步地执行block。示例中的aBlock变量定义的block类型为，接收单个int型参数并且没有返回值。代码中生成了一个符合该类型的内联block，并赋给了aBlock。最后一行直接执行了block。

代码清单3-1 简单block示例
```
int x = 123;
int y = 456;
 
// Block declaration and assignment
void (^aBlock)(int) = ^(int z) {
    printf("%d %d %d\n", x, y, z);
};
 
// Execute the block
aBlock(789);   // prints: 123 456 789
```
接下来的是一些设计block的注意点：
- 对于你要在派发队列中异步执行的block来说，从父域中捕获标量型变量是安全的。但是，不要捕获父域中由调用环境【calling context】创建并销毁的结构体或其他指针型变量。否则，当你的block执行时，该指针引用的内存已经非法了。当然，你自己创建的内存（或对象），以及显式将拥有关系转交给block的，都是安全的。
- 派发队列会复制提交给它的block，并且执行完后释放block。换句话说，你在将block提交到队列时不必显式复制。
- 尽管使用队列执行小任务比使用线程高效。但创建block和在队列中执行还是有开销的。如果block中工作过于少，可能内联执行【execute it inline】要比提交给队列更合适。想要确定任务的工作是不是太少，可以使用相关的性能工具测量后分析。
- 不要利用底层线程缓存数据，并指望其他blcok可以访问这些数据。如果同一个队列中的任务想要共享数据，使用派发队列的上下文指针【context pointer】来存储数据。更多信息请参考后文的“存储自定义的队列上下文信息”
- 如果block创建了很多OC对象。你可以在block中使用@autorelease块来优化内存管理。尽管GCD派发池有自己的自动释放池，但什么时候清空池并不确定。如果你的程序有内存限制，就要创建自己的自动释放池来加速释放对象。

更多关于block的信息，请参考[Blocks Programming Topics](https://developer.apple.com/library/content/documentation/Cocoa/Conceptual/Blocks/Articles/00_Introduction.html#//apple_ref/doc/uid/TP40007502)文档。更多关于将block加入队列的信息，请参考后文的“向队列添加任务”一节。

### 创建和管理派发队列
将任务添加到队列之前，你首先要决定使用什么队列。派发队列可以是串行的也可是并行的。除此之外，如果你的队列有特定的需求，还可以配置一些其他属性。接下来的小节讲述了如何创建及配置队列。

#### 获取全局并发派发队列
如果你有多个任务可以并行执行，那么你可以使用并发队列。并发队列也使用先进先出的策略，但它允许后面的任务不需等待前面的任务执行完毕，就可以开始执行。并行的任务数会根据系统条件动态变化。很多因素会影响并发数，例如CPU的可用核心数，其他进程的任务量，以及其他串行派发队列中任务的数量及优先级。

系统为每个程序提供了4个并行队列。这4个队列是全局的，它们的优先级各不相同。你无需显式地创建它们，直接使用dispatch_get_global_queue函数获取它们即可，如下所示：

```
dispatch_queue_t aQueue = dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0);
```
你可以通过传递参数获取不同优先级的队列。DISPATCH_QUEUE_PRIORITY_HIGH用来获取高优先级，DISPATCH_QUEUE_PRIORITY_LOW用来获取低优先级，DISPATCH_QUEUE_PRIORITY_BACKGROUND用来获取后台队列。如你所料，高优先级队列中的任务比默认级别队列中的任务更早执行，默认级别的比低级别的更早执行。

> 注意：dispatch_get_global_queue函数的第2个参数是为以后扩展保留的，目前传0即可。

尽管派发队列是引用计数对象，但你不需要对全局队列进行retain和release。它们是程序全局的，对它们retain和release会被忽略。因此，你也不需要存储它们的引用，使用时通过dispatch_get_global_queue获取即可。

#### 创建串行队列
如果你想让任务按照特定顺序执行，你需要使用串行队列。串行队列每次只执行一个任务。你也可以使用串行队列代替锁，来控制共享资源的访问。和锁不一样的是，串行队列可以包含任务的执行顺序。只要你异步地向串行队列提交任务，就不会产生死锁。

和并发队列不同的是，系统没有为你创建全局的串行队列，你必须显式地创建和管理串行队列。不要为了并发处理大量任务而创建大量的串行队列，这种需求应该使用并发队列。创建串行队列之前，要思考使用它的目的，例如控制共享资源的访问，或同步某些关键操作。

代码清单3-2展示了如何创建串行队列。dispatch_queue_create函数有两个参数，前者是队列名，后者是一组队列特性。队列名通常在你使用调试工具和性能工具时让队列易于识别。队列特性是为了扩展预留的字段，传NULL即可（译注：目前已经可以使用，例如指定串行还是并行）。

代码清单3-2 创建串行队列
```
dispatch_queue_t queue;
queue = dispatch_queue_create("com.example.MyQueue", NULL);
```
除了你自己创建的串行队列之外，系统还会自动创建一个串行队列并绑定到程序的主线程。更多信息请参考后文“在运行期获取通用队列”一节。

#### 在运行期获取通用队列【Common Queues】
GCD提供了一些函数供你在访问几个通用的派发队列。
- 使用dispatch_get_current_queue可获取当前队列，一般是处于调试或测试的目的才使用这个函数。在block内部调用此函数，会返回当前block所在的队列，在block外部调用此函数，会返回系统默认的并发队列【the default concurrent queue】。
- 使用dispatch_get_main_queue来获取和主线程绑定的串行队列。Cocoa程序会自动创建此队列，其他程序如果调用dispatch_main函数或为主线程配置Runloop（使用CFRunLoopRef或NSRunloop），也会自动创建此队列。
- 使用dispatch_get_global_queue函数获取程序级的共享并发队列。更多信息请参考前文的“获取全局并发队列”一节。

### 派发队列的内存管理
派发队列和其他GCD对象都是采用引用计数机制。当你创建串行队列时，它的初始引用计数为1.你可以使用dispatch_retain和dispatch_release函数来增加或减少它的引用计数。当引用计数减为0时系统会异步的销毁【deallocate】这个队列。

retain和release GCD对象很重要。例如队列，如果你要使用它，就要保证它还在内存中。Cocoa对象的内存管理，一个通用的规则是如果你在使用一个对象前retain它，使用完毕后release它。遵循这个模式可以保证当你使用队列时，它还在内存中。

> 注意：你不需要retain和release全局队列，不管是全局并发队列还是主队列。对它们调用retain和release会被忽略。

即使你的程序是在垃圾回收机制下，你依然要retain和release GCD对象。GCD不支持垃圾回收机制。

#### 存储队列的自定义上下文信息【Storing Custom Context Information with a Queue】
所有GCD对象【dispatch objects】，包括派发队列，都支持你绑定自定义的上下数据。你可以使用dispatch_set_context和dispatch_get_context这两个函数来设置或读取数据。系统不会使用你创建的数据，你有责任在适当的时候创建和销毁数据。

对于队列来说，你可以使用上下文数据来存储OC对象或其他结构体的指针，以此来识别队列，或满足你的其他需求。你可以使用队列的终结函数【finalizer function】来在队列被销毁前，先销毁它的上下文数据。如何使用终结函数清理上下文数据请参考后文的代码清单3-3。

#### 为队列提供清理函数
你创建队列后（译注：原文是串行队列，但文档书写时只能创建串行队列，故此处使用队列一词），可以为其添加终结函数，来在队列被销毁前，先执行自定义的清理工作。你可以给使用dispatch_set_finalizer_f函数给队列指定终结函数，队列引用计数为0时，终结函数会执行。你可以使用这个函数来清理上下文数据，只有上下文指针不为NULL时，这个函数才会调用。
代码清单3-3展示如何创建一个终结函数并将其指定给队列。队列使用终结函数来释放上下文指针中的数据。（myInitializeDataContextFunction和myCleanUpDataContextFunction是你自定义的，如何初始化和清理数据结构内容的函数）。终结函数接到的上下文参数中包含和队列绑定的数据。

代码清单3-3 为队列添加终结函数
```
void myFinalizerFunction(void *context)
{
    MyDataContext* theData = (MyDataContext*)context;
 
    // Clean up the contents of the structure
    myCleanUpDataContextFunction(theData);
 
    // Now release the structure itself.
    free(theData);
}
 
dispatch_queue_t createMyQueue()
{
    MyDataContext*  data = (MyDataContext*) malloc(sizeof(MyDataContext));
    myInitializeDataContextFunction(data);
 
    // Create the queue and set the context data.
    dispatch_queue_t serialQueue = dispatch_queue_create("com.example.CriticalTaskQueue", NULL);
    if (serialQueue)
    {
        dispatch_set_context(serialQueue, data);
        dispatch_set_finalizer_f(serialQueue, &myFinalizerFunction);
    }
 
    return serialQueue;
}
```

### 向队列添加任务
想要执行任务，你要先把它添加到合适的队列中。添加可以是同步的，也可以是异步的，可以是添加单个任务，也可以是添加一组任务。任务一旦加入队列，队列就会尽可能早的执行它，执行时机取决于队列本身的约束，以及队列中其他任务的情况。本节展示了向队列添加队伍中的技术点，并描述了这些技术点的优势。

#### 向队列添加单个任务
添加任务有两种方式，同步的或异步的。通常来讲，异步执行（使用 dispatch_async或dispatch_async_f函数）要优于同步执行。任务加入队列后，你无法得知任务开始执行的准确时间。异步执行任务可以让相关代码异步执行，以保证你的调用线程可以继续工作，不必等待。如果调用线程是主线程（例如为了响应用户操作），那么异步执行更加重要。

虽然通常你要异步执行任务，但有些时候，还是需要同步执行，例如你需要避免竞争条件【race conditions】或其他同步问题。这种情况下，你可以使用dispatch_sync或dispatch_sync_f来同步执行任务。这些函数会阻塞调用线程，直到任务执行完毕。

> 重要：不要在任务中使用dispatch_sync或dispatch_sync_f来向本任务所在的队列中添加任务。在串行队列，这将造成死锁，在并发队列中，也要尽量避免这种用法。

下面的代码展示了如何通过block方式添加异步任务和同步任务：
```
dispatch_queue_t myCustomQueue;
myCustomQueue = dispatch_queue_create("com.example.MyCustomQueue", NULL);
 
dispatch_async(myCustomQueue, ^{
    printf("Do some work here.\n");
});
 
printf("The first block may or may not have run.\n");
 
dispatch_sync(myCustomQueue, ^{
    printf("Do some more work here.\n");
});
printf("Both blocks have completed.\n");

```

#### 任务完成后执行完成块【Completion Block】
通常，队列中的任务独立地执行代码，和创建任务的代码无关。但有时候，创建任务的代码想要知道任务何时执行完毕，或想使用任务结束时得到的结果数据，这种情况下，程序应该能在任务结束时通知创建任务的代码。传统的异步编程中，一般使用回调函数，在派发队列中，你可以使用完成块。

完成块就是在任务结束后执行的一段代码。通常，调用方代码启动任务时，会将完成块作为一个参数。任务执行完毕后要执行的代码，都包含在完成块中。

代码清单3-4展示了使用block实现的函数。后两个参数允许调用方指定完成块及完成快执行的队列。函数计算完结果后，会在给定的队列中执行完成块。为了避免队列被释放，函数先retain了队列，在完成块被加入该队列后又release了队列。

代码清单3-4 任务完成后执行完成块
```
void average_async(int *data, size_t len,
   dispatch_queue_t queue, void (^block)(int))
{
   // Retain the queue provided by the user to make
   // sure it does not disappear before the completion
   // block can be called.
   dispatch_retain(queue);
 
   // Do the work on the default concurrent queue and then
   // call the user-provided block with the results.
   dispatch_async(dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0), ^{
      int avg = average(data, len);
      dispatch_async(queue, ^{ block(avg);});
 
      // Release the user-provided queue when done
      dispatch_release(queue);
   });
}
```

#### 并发地执行迭代【Performing Loop Iterations Concurrently】
有一种场景，并发队列可以提高性能，这种场景是在确定次数的迭代中执行任务。例如在for循环中执行任务。
```
for (i = 0; i < count; i++) {
   printf("%u\n",i);
}
```
如果迭代中执行的任务是相互独立的，并且执行顺序是随意的，那么可以使用dispatch_apply或dispatch_apply_f来取代传统迭代。这个函数可以在每个迭代向并发队列提交一个任务，这样迭代中的任务就可以并发执行。

你可以在串行队列或并行队列中使用这个函数，但通常用在并发队列中。虽然在串行队列中可以使用，但对性能提升没有作用。

> 重要：和for循环一样，dispatch_apply或dispatch_apply_f函数直到迭代完成才会返回。当在队列中执行的任务中调用这些函数要小心。如果在串行队列的任务中，调用apply函数时，向其传入的队列是这个串行队列中，会造成死锁。（译注：推测，apply在指定的队列上sync执行迭代，因此会死锁，迭代中加入队列的任务是aync的，故apply迭代完毕即可返回）。
考虑到apply函数会阻塞当前线程，在主线程中使用该函数时要小心。如果你的迭代操作比较耗时，应该考虑将apply调用放到其他线程。


代码清单3-5展示了使用dispatch_apply替代for循环。函数第一个参数count是迭代次数，函数的最后一个参数是任务block，其唯一参数是当前迭代的索引，从0到count-1。

代码清单3-5 并发执行迭代
```
dispatch_queue_t queue = dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0);
 
dispatch_apply(count, queue, ^(size_t i) {
   printf("%u\n",i);
});
```

你应该确保任务的大小和数量是合理的。任务注册及执行，有额外的开销。如果迭代中注册的任务数量巨大，但每个任务执行的代码很少，那么额外的开销就比较显著。如果测试中确实发现此问题，你可以使用跨步【striding】的方式增加每次迭代周期的工作，你可以将几个迭代周期中的任务合并为一个任务，以减少迭代次数。例如，调整前迭代数为100，使用4的跨度，调整后迭代数为25。更多信息请参考[Improving on Loop Code](https://developer.apple.com/library/content/documentation/General/Conceptual/ConcurrencyProgrammingGuide/ThreadMigration/ThreadMigration.html#//apple_ref/doc/uid/TP40008091-CH105-SW2)。

#### 在主线程执行任务
GCD提供了一个特殊的派发队列，允许你将任务派发到主线程执行（译注：后文简称主队列）。应用会自动创建主队列，并且会被主线程的Runloop（CFRunLoopRef或NSRunLoop对象）自动清空【drained】（译注：根据上下文，清空的意思应该是将任务出队列执行）。如果你的应用**不是**Cocoa应用，并且不想显式创建Runloop，那么你需要调用dispatch_main函数来显式清空主队列。如果这种场景下你依然向队列中添加任务，不调用此函数的话，任务不会执行。

#### 在任务中使用OC对象
GCD支持Cocoa的内存管理，你可以在提交到队列中的任务Block中使用OC对象。每个派发队列管理自己的自动释放池，在恰当的时机释放自动释放对象，队列不保证释放对象的准确时机。
如果应用的内存受限，并且你的block中创建了大量自动释放对象，你需要自己创建自动释放池。如果自动释放对象数量很多，你可以要创建多个自动释放池，或周期性的清空自动释放池。

更多自动释放池和OC内存管理的信息，请参考[Advanced Memory Management Programming Guide](https://developer.apple.com/library/content/documentation/Cocoa/Conceptual/MemoryMgmt/Articles/MemoryMgmt.html#//apple_ref/doc/uid/10000011i)文档。

### 挂起和恢复队列
你可以挂起队列来临时阻止队列执行任务。使用dispatch_suspend函数来挂起队列，使用dispatch_resume函数来恢复队列。调用dispatch_suspend会增加队列的挂起引用计数，调用dispatch_resume会减少该计数。挂起引用计数大于0时，队列会保持挂起状态。因此，你要保持dispatch_suspend和dispatch_resume的一一对应。

>  重要：挂起和恢复调用都是异步的，在任务执行的间隙生效【 take effect only between the execution of blocks】。挂起队列不会影响正在执行的任务。

### 使用派发信号量协调共享资源【 Finite Resources】
如果队列的任务需要访问共享资源【finite resource】，你可能需要使用派发信号量来协调多个任务同步地访问该资源。派发信号量和传统信号量类似，但它的优势在于，如果资源可用的话，它能更快地获取【acquire】信号量，因为GCD在这种情况下不会产生内核调用。只有资源不可用时，才会产生内核调用，系统会暂停【part】该线程直到信号量接到通知【until the semaphore is signaled】。
使用派发信号量的语义如下：
1. 创建信号量时（使用dispatch_semaphore_create函数），你可以指定一个正整数来指定可用资源数。
2. 每个任务，都调用dispatch_semaphore_wait来等待信号量。
3. dispatch_semaphore_wait返回时，获取资源并执行具体工作。
4. 使用完资源后，释放资源并且调用dispatch_semaphore_signal通知信号量。

为了展示这个流程，我们假设要使用系统的文件描述符。每个程序可以使用有限数量的文件描述符。如果有个任务要处理大量的文件，同一时间打开这些文件不合适，可能会耗尽文件描述符。相反，你可以使用信号量来限制同一时间使用的文件描述符的数量，基本的代码如下所示：
```
// Create the semaphore, specifying the initial pool size
dispatch_semaphore_t fd_sema = dispatch_semaphore_create(getdtablesize() / 2);
 
// Wait for a free file descriptor
dispatch_semaphore_wait(fd_sema, DISPATCH_TIME_FOREVER);
fd = open("/etc/services", O_RDONLY);
 
// Release the file descriptor when done
close(fd);
dispatch_semaphore_signal(fd_sema);
```
创建信号量时，你指定可用资源的数量。这个值就是信号量的初始可用数量。每次你调用dispatch_semaphore_wait时，可用数量会减1，如果可用数量为负值（译注：文档中没提到0的情况？），函数会告知内核阻塞该线程。反之，调用dispatch_semaphore_signal时，可用数量会加1，以此来指示有一个资源可用，如果此时有正在等待资源的阻塞线程，它会解除阻塞继续工作。

### 等待一组任务执行完成【Waiting on Groups of Queued Tasks】
派发组【Dispatch groups】技术可以阻塞一个线程，直到目标任务或一组任务执行完毕。在需要等待任务或一组任务执行完毕的场景，你可以使用这项技术。例如，几个任务在执行计算任务，你的任务需要等待它们都计算完毕，使用它们的计算结果来进行后续处理。另一个使用场景，是使用该技术来代替线程join。使用传统线程，是创建几个子线程，并使用join将其关联。而使用派发组，则是向派发组中添加一组任务，并等待整个组执行完毕。

代码清单3-6展示了派发组技术的基本流程：建立组、向组派发任务、等待结果。使用dispatch_group_async向组派发任务，此函数会将任务和组及执行队列绑定。使用dispatch_group_wait来等待特定的组任务执行完成。

代码清单3-6 等待一组异步任务
```
dispatch_queue_t queue = dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0);
dispatch_group_t group = dispatch_group_create();
 
// Add a task to the group
dispatch_group_async(group, queue, ^{
   // Some asynchronous work
});
 
// Do some other work while the tasks execute.
 
// When you cannot make any more forward progress,
// wait on the group to block the current thread.
dispatch_group_wait(group, DISPATCH_TIME_FOREVER);
 
// Release the group when it is no longer needed.
dispatch_release(group);
```

### 派发队列和线程安全
在派发队列的文章中讨论线程安全看上去有点奇怪【odd】，不过线程安全是个有意义的话题。当你处理并发时，以下几点需要注意：
- 派发队列是线程安全的，这意味你可以在任意线程向队列提交任务，不要加锁或作其他同步处理。
- 不要在队列中的任务中使用dispatch_sync向该队列提交任务，可能造成死锁，如果要提交，使用dispatch_async。
- 避免在队列的任务中使用锁。尽管在任务中使用锁是安全的，但当你的线程在未获取到锁时，线程会被阻塞。如果是串行队列，整个队伍都会被阻塞，即便是并发队列，阻塞线程也会影响其他任务。如果需要同步的话，使用串行队列代替锁。
- 尽量避免从底层线程获取信息。更多关于派发队列与线程的兼容性，请参考[Compatibility with POSIX Threads](https://developer.apple.com/library/content/documentation/General/Conceptual/ConcurrencyProgrammingGuide/ThreadMigration/ThreadMigration.html#//apple_ref/doc/uid/TP40008091-CH105-SW18)一节。

更多关于从传统线程向派发队列迁移的信息，请参考[Migrating Away from Threads](https://developer.apple.com/library/content/documentation/General/Conceptual/ConcurrencyProgrammingGuide/ThreadMigration/ThreadMigration.html#//apple_ref/doc/uid/TP40008091-CH105-SW1)一章。



