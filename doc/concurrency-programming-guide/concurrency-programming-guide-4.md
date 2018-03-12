[Concurrency Programming Guide 官档传送门](https://developer.apple.com/library/content/documentation/General/Conceptual/ConcurrencyProgrammingGuide/Introduction/Introduction.html#//apple_ref/doc/uid/TP40008091)

本文翻译自2012-12-13版的官档。
您可以在官档结尾的Document Revision History中查阅版本。
注：【】包含的部分引自原文。

## 从线程迁移
有很多方式可将现有的线程代码迁移至GCD和operation。尽管不是所有的场景都能去除线程代码，但可以去除的部分能显著提高性能（以及代码的简洁性）。使用派发队列和operations替换线程有下述优势：
- 应用不再为线程栈提供内存
- 去除了创建和配置线程的代码
- 去除了向线程注册任务及管理的代码
- 简化了代码

如何使用派发队列和操作队列替换掉基于线程的代码，本章提供了相关提示和指南。

### 使用派发队列替换线程
要理解如何使用派发队列替换线程，首先要考虑程序中通常如何使用线程：
- 单任务线程。创建线程执行单个任务，任务结束后释放线程。
- 工作者【Worker】线程。创建一个或多个针对特定任务的工作者线程。周期性地向工作者线程派发任务。
- 线程池。创建通用线程池，并且为每个线程设置runloop。当有任务要执行时，从池中获取一个线程，将任务派发给它。如果没有空闲线程，任务在队列等待线程可用。

尽管这些看上去是不同的技术，但它们实质上是一样的：程序将任务交给线程执行。它们的区别只在于管理线程及队列化任务的代码。使用派发队列，你可以去除这些线程相关代码，从而将精力集中到具体业务上。

如果你在使用上述的线程技术，你对程序中的任务是很了解的。和之前将任务提交给线程不同，现在你要将任务封装到operation对象或block中，将其提交到合适的队里。对于没用同步要求的任务（不需要锁的任务），你可以按下述方式直接替换：
- 对单任务进程，将任务封装入operation对象或block中，提交给并发队列。
- 对于工作者进程，你要决定使用串行队列还是并发队列。如果你使用工作者线程来同步一组任务，那么使用串行队列。如果要执行互不依赖的独立任务，使用并发队列。
- 对于线程池，将任务封装入operation对象或block中，提交给并发队列。

当然，不是所有的情况都可以这么简单替换。如果任务要访问共享资源，最理想的方案是移除掉或最小化共享资源。如果通过重构代码可以消除共享资源，是最好的。如果不行的话，那么就是队列提供的技术。派发队列的一大优点就是代码更有预期性。预期性体现在，即使不使用锁或其他重型同步机制，也可以同步代码。你可以使用如下方式来同步任务：
- 如果任务依赖特定的执行顺序，那么将它们按顺序提交到串行队列。如果你想使用操作队列，那么为每个任务设置依赖，来确定任务按特定顺序执行。
- 如果任务要访问共享资源，那么将相关任务提交到串行队列中。串行队列可以取代锁或其他同步机制。更多信息可参考后文的“去除基于锁的代码”。
- 如果线程要等待其他线程上任务执行完毕，可以考虑用派发队列的组【groups】来实现。如果你想使用操作队列，也可以为任务添加依赖开模拟出组的效果。更多信息，请参考后文的“替换线程Join”一节。
- 如果你使用了生产者-消费者算法来管理受限资源池，可以参考后文的“改变生产者-消费者的实现”。
- 如果你使用线程来读写描述符或监视文件操作，可以使用派发源，请参考[Dispatch Sources](https://developer.apple.com/library/content/documentation/General/Conceptual/ConcurrencyProgrammingGuide/GCDWorkQueues/GCDWorkQueues.html#//apple_ref/doc/uid/TP40008091-CH103-SW1)一章。

要注意的是，队列不是替换线程的万金油。队列提供的异步编程模型通常用于延迟不敏感的场景。尽管任务有优先级的概念，但不能保证在确定的时间执行任务。因此，在延迟敏感的场景（如音视频的播放），线程可能是更好的选择。

### 消除基于锁的代码
对于线程代码，传统的同步方式之一是使用锁。锁的开销是比较大的。即使在非竞争【non-contested】的情况下（译注：推测，是共享资源当前可用的情况），锁相关的代码也有一定的开销。在竞争的情况下，可能会有多个线程为等待资源而被阻塞一段时间。
使用队列替换基于锁的代码，可以去除锁相关的代码，并能简化其他部分的代码。你可以使用串行队列来控制共享资源的访问。串行队列比锁的开销少，例如，串行队列的任务不需要
调用内核去获取互斥锁【mutex】。
使用队列，你需要决定是同步执行任务，还是异步执行任务。提交异步任务可以保持当前线程继续运行，提交同步任务会阻塞当前线程直到任务执行完毕。两者都有合适的场景，但异步任务更常用。
接下来的小节展示了如何使用队列替换基于锁的代码。

#### 实现异步锁
异步锁是一种非阻塞的保护共享资源的方式，你可以安全的使用共享资源，不会阻塞其他想使用共享资源的代码。当你的代码修改数据结构会影响其他代码时，你应该使用异步锁。使用传统线程，你通常是这样实现的：获取共享资源的锁、执行必要的操作、释放锁、继续主要的工作。然后，使用派发队列的话，你的代码可以异步操作，不需要等待操作完成。

代码列表5-1展示了异步锁的实现。示例中，共享资源定义了自己的串行队列，访问方向该队列提交一个block任务。由于队列是串行的，它会按任务提交顺序访问资源。由于任务是异步执行的，访问线程不会被阻塞。

代码列表5-1 异步修改共享资源
```
dispatch_async(obj->serial_queue, ^{
   // Critical section
});
```

#### 同步操作临界区
如果当前代码需要等待某个任务完成才能执行，你可以使用dispatch_sync来提交同步任务。此函数向派发队列提交任务，然后阻塞本线程，直到任务执行完毕。派发队列可以是并发的也可以是串行的。由于函数会阻塞线程，因此只在必要的时候使用它。代码列表5-2展示了使用dispatch_sync来包装一个临界区。

代码列表5-2 同步运行临界区
```
dispatch_sync(my_queue, ^{
   // Critical section
});
```

如果你已经使用串行队列来保护共享资源，那么向队列提交同步任务不会比异步任务带来更好的保护性。提交同步任务的唯一理由应该是在临界区没执行完毕前阻塞当前代码。例如，如果你想从共享资源中获取数据并马上使用，那么应该使用同步任务。如果当前代码不需要等待临界区完成，或者它能向同一队列提交后续的任务【or if it can simply submit additional follow-up tasks to the same serial queue】，那么异步任务是更好的选择。

### 改进循环代码
如果你的代码中有循环，并且循环中的工作是独立的，你可以使用dispatch_apply或dispatch_apply_f来重新实现循环。这些函数会每个迭代作为任务提交到队列中。如果使用并发队列，这个功能可以并发的执行多个迭代任务。
dispatch_apply和dispatch_apply_f是同步函数，调用会阻塞当前线程，直到循环完成。使用并发队列时，迭代执行的顺序没有保障。执行每个迭代任务的线程可能会阻塞，导致某个迭代任务可能先于或后于相邻的迭代任务完成，因此，每个迭代任务都应该是可重入的【reentrant】（译注：此处没理解，为什么需要可重入）。
代码列表5-3展示了如何使用dispatch_apply替换for循环。传给dispatch_apply或dispatch_apply_f的整型参数需要指定迭代次数。本例是简单的打印迭代索引。

代码列表5-3 替换for循环
```
queue = dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0);
dispatch_apply(count, queue, ^(size_t i) {
   printf("%u\n", i);
});
```

尽管上面的例子很简单，但它展示了如何使用派发队列替换基本循环。尽管apply可以有效的提高性能，但你依然要有选择的使用。派发队列的开销不大，但依然有开销，例如向线程注册任务。因此，你要确保你的迭代任务中做了足够多的工作，来让这些额外开销显得微不足道。多少工作合适，要根据性能工具的分析结构决定。
跃进【striding】是一种提升迭代任务工作量的方式。使用跃进，你可以重写代码，让一个迭代任务执行原循环中几个迭代的任务。这样就可以减少apply的迭代任务数。代码清单5-4展示了使用跃进实现5-3中的代码。在5-4中，block中多次执行打印。由于有余数，余数部分是内联执行的。

代码列表5-4 跃进方式执行迭代任务
```
int stride = 137;
dispatch_queue_t queue = dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0);
 
dispatch_apply(count / stride, queue, ^(size_t idx){
    size_t j = idx * stride;
    size_t j_stop = j + stride;
    do {
       printf("%u\n", (unsigned int)j++);
    }while (j < j_stop);
});
 
size_t i;
for (i = count - (count % stride); i < count; i++)
   printf("%u\n", (unsigned int)i);
```

使用跃进会有一定的性能提升。实践中，迭代次数过多的场景，使用跃进有优势。更少的迭代次数意味着大部分时间用在执行业务上而不是用在派发任务上。你应该根据性能分析结果找到合适的跃进值。

### 替换线程连接【Join】
线程连接允许你生成其他线程，当前线程等其他线程完成后才继续工作。为了实现线程连接，父线程创建一个可连接【joinable】子线程。当父线程需要子线程的某个执行结果才能继续的时候，它会采用连接的方式。连接会阻塞父线程，直到子线程完成任务并退出，此时父线程拿到处理结果，继续进行自己的工作。如果父线程需要连接多个子线程，它同一时间只能执行一个连接。
派发组【Dispatch group】提供了和连接类似的功能，但有更多优势。和连接类似，派发组也是阻塞线程直到一个或多个子任务执行完毕。和连接不同的是，派发组可以同时等待多个子任务执行。由于派发组是使用派发队列执行队列，因此性能更好。

使用派发组替换连接，你可以按如下步骤操作：
1. 使用dispatch_group_create创建派发组。
2. 使用dispatch_group_async或dispatch_group_async_f来将任务添加到组。任务即是你之前想在子线程上处理的工作。
3. 在当前线程上需要等待组任务执行完毕的点调用dispatch_group_wait。这个函数会阻塞当前线程，知道组内任务全都执行完毕。

如果你使用operation对象来实现任务。你可以使用依赖来替代连接。和父线程等待子任务执行完毕不同的是，你需要将父线程代码也封装成operation对象，然后将其设置为依赖子任务operation，这样在只有被依赖的子任务执行完毕后，父任务才会执行。

如果想参考派发组的示例，请参考[Waiting on Groups of Queued Tasks](https://developer.apple.com/library/content/documentation/General/Conceptual/ConcurrencyProgrammingGuide/OperationQueues/OperationQueues.html#//apple_ref/doc/uid/TP40008091-CH102-SW25)章节，更多关于operation依赖的信息，请参考[Configuring Interoperation Dependencies](https://developer.apple.com/library/content/documentation/General/Conceptual/ConcurrencyProgrammingGuide/OperationObjects/OperationObjects.html#//apple_ref/doc/uid/TP40008091-CH101-SW17)章节。

### 改变生产者-消费者的实现
你可以使用生产者-消费者模型来管理一组动态生成的有限资源。生产者生成新资源（或工作）的同事，一个或多个消费者等待资源（或）工作就绪以便消费。典型的生产者-消费者机制是条件【conditions】或信号量【semaphores】。
使用条件的话，生产者线程通常如下操作：
1. 使用pthread_mutex_lock给条件的互斥量加锁。
2. 生成资源（要执行的工作）。
3. 使用pthread_cond_signal提醒条件变量已有可供消费的资源。
4. 使用pthread_mutex_unlock给互斥量解锁。
相应的，消费者线程如下操作：
1. 使用pthread_mutex_lock为条件的互斥量加锁。
2. 建立while循环：
    1. 检查是否有资源（工作）。
    2. 如果没有资源（工作），调用pthread_cond_wait阻塞当前线程直到有相关的通知。
3. 获取生产者提供的资源（工作）。
4. 使用pthread_mutex_unlock给互斥量解锁。
5. 使用资源（执行工作）。

在派发队列里，你可以简单使用一个调用实现，如下：
```
dispatch_async(queue, ^{
   // Process a work item.
});
```
当生产者产生资源（要执行的任务）时，它要做的就是将任务加入队列，让队列处理它们（译注：此处队列就相当于消费者）。你唯一要做的就是决定队列的类型。如果任务需要特定的顺序，就使用串行队列。如果任务可以并发进行，就使用并发队列。

### 替换信号量代码
如果当前代码使用信号量来控制共享资源的访问，你可以使用派发信号量【dispatch semaphores】替换。传统信号量经常调用内核来检查信号量，而派发信号量在用户空间更为快速的检查信号量，只有检查失败并且线程需要阻塞时才会调用内核。这就导致了派发信号量在资源非竞争状态下比传统信号量更为快速。除此之外，两者行为是类似的。
如果需要派发信号量的示例代码，请参考[Using Dispatch Semaphores to Regulate the Use of Finite Resources](https://developer.apple.com/library/content/documentation/General/Conceptual/ConcurrencyProgrammingGuide/OperationQueues/OperationQueues.html#//apple_ref/doc/uid/TP40008091-CH102-SW24)章节。

### 替换Runloop代码
如果你使用runloop来管理线程上的工作，你可能会感觉使用队列能简化这些工作。建立自定义的runloop涉及到建立底层的线程和runloop本身。runloop代码包含建立一个或多个runloop源，以及实现事件处理的回调代码。为了取代这些工作，你可以简单的创建一个串行队列，然后向队列提交队列。上面的操作只需要一行代码：
```
dispatch_queue_t myNewRunLoop = dispatch_queue_create("com.apple.MyQueue", NULL);
```
由于队列会自动执行添加的任务，因此你不用写额外的代码来管理队列。你不需要创建和配置线程，也不需要创建和挂接runloop源。另外，如果你想执行新类型的任务，简单的向队列添加新任务就可以了。如果使用runloop的话，要执行新类型任务，你还需要修改现存的runloop源或创建的新的runloop源。

一个常见的runloop配置是处理网络socket异步获取的数据。你可以将一个派发源挂接到合适队列来取代配置runloop。相比传统的runloop源，派发源能提供更多的配置项【options】。除了处理timer和网络端口事件，你还可以使用派发源读写文件、监视文件系统对象、监视进程以及监视系统信号。你还可以自定义派发源，在你代码中特定的时机异步触发。更多信息请参考[Dispatch Sources](https://developer.apple.com/library/content/documentation/General/Conceptual/ConcurrencyProgrammingGuide/GCDWorkQueues/GCDWorkQueues.html#//apple_ref/doc/uid/TP40008091-CH103-SW1)一章。

### 和POSIX线程兼容
GCD会管理你提供的任务和执行任务的底层线程，因此你通常应该避免在任务中调用POSIX线程的例程【routines】。如果确实需要调用的话，你应该非常小心。本节介绍了哪些例程在任务中调用是安全的，哪些是不安全的。这个清单并不完整，但能做基本的指导。

通常，你的应用不应该修改或删除那些不是由你生成对象。因此，你的任务中不能调用下述函数：
pthread_detach
pthread_cancel
pthread_join
pthread_kill
pthread_exit

你可以在任务中修改线程状态，但在任务返回前，要恢复线程的状态。因此，你可以调用下述函数，但要记得在返回前恢复线程的状态：
pthread_setcancelstate
pthread_setcanceltype
pthread_setschedparam
pthread_sigmask
pthread_setspecific

执行任务的底层线程是任务执行之间可能变化的【from invocation to invocation】（译注：未理解，推测是执行同一任务时不会切换线程）。因此，你的程序不应该依赖block调用间下述函数返回的结果【your application should not rely on the following functions returning predictable results between invocations of your block】：
pthread_self
pthread_getschedparam
pthread_get_stacksize_np
pthread_get_stackaddr_np
pthread_mach_thread_np
pthread_from_mach_thread_np
pthread_getspecific

> 任务【block】必须捕获并消除【suppress】任务中发生的语言级的异常。任务执行中出现的其他错误应该处理掉或通知程序的其他部分。

更多关于POSIX线程的信息，请参考pthread man页面。

