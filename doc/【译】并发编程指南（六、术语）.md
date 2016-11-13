[Concurrency Programming Guide 官档传送门](https://developer.apple.com/library/content/documentation/General/Conceptual/ConcurrencyProgrammingGuide/Introduction/Introduction.html#//apple_ref/doc/uid/TP40008091)

本文翻译自2012-12-13版的官档。
您可以在官档结尾的Document Revision History中查阅版本。
注：【】包含的部分引自原文。

## 术语
（译注：本节介绍术语，强烈建议阅读原文[Glossary](https://developer.apple.com/library/content/documentation/General/Conceptual/ConcurrencyProgrammingGuide/Glossary/Glossary.html#//apple_ref/doc/uid/TP40008091-CH104-SW2)）
- 应用【application】。 程序【program】的类型之一，可以像用户展示图形界面。
- 异步设计方式【asynchronous design approach】。一种组织应用的原则，程序包含很多代码块，可以并发的在应用的主线程或其他线程上执行。由一个线程启动的异步任务，可以在其他线程上执行实际的工作，可以利用额外的处理器资源来更快的完成工作。
- block【block object】。一种C的结构体，它封装了内联的代码和数据，以便于在后续的执行中使用。你可以使用block来封装任务，或者内联的在当前线程执行，也可以利用派发队列在其他线程执行。可参考[Blocks Programming Topics](https://developer.apple.com/library/content/documentation/Cocoa/Conceptual/Blocks/Articles/00_Introduction.html#//apple_ref/doc/uid/TP40007502)。
- 并发operation【concurrent operation】。一种operation对象，它不在调用start方法的线程上执行。并发operation会建立自己的线程，或通过接口获取其他线程，来执行自己的工作。
- 条件【condition】。一种用于同步访问资源的结构。正在等待条件的线程不允许处理工作，知道其他线程显式通知【signal】条件。
- 临界区【critical section】。同一时间只能被一个线程执行的一段代码。
- 自定义源【custom source】。派发源可用于处理应用自定义的事件。应用产生事件后，自定义源可以调用相关的自定义事件处理器处理事件。
- 描述符【descriptor】。一种抽象的标识符，用于访问文件、socket或其他系统资源。
- 派发队列【dispatch queue】。一种GCD结构，你可以使用它执行应用的任务。GCD定义了串行和并发两种队列。
- 派发源【dispatch source】。一种GCD结构，你可以使用它来处理系统相关事件。
- 描述符派发源【descriptor dispatch source】。一种派发源，用于处理文件相关的事件。文件描述符源可以在文件可读、可写、有变换时产生事件，供你自定义的事件处理器处理。
- 动态库【dynamic shared library】。一种可执行的二进制库。它会被动态的加载到应用的进程空间，而不是静态链接到应用的可执行程序中。
- 库【framework】（译注：翻译成框架更符合这个单词，但在翻译时可能有误导作用）。一种包【bundle】，它将动态库和相关的头文件、资源打包在一起。更多信息，请参考[Framework Programming Guide](https://developer.apple.com/library/content/documentation/MacOSX/Conceptual/BPFrameworks/Frameworks.html#//apple_ref/doc/uid/10000183i)文档。
- 全局派发队列【global dispatch queue】。GCD为你的应用自动创建的派发队列。你不需要自己创建全局派发队列，也不需要retain和release。你可以使用系统提供的函数获取这些队列。
- GCD【Grand Central Dispatch】。一种并发执行异步任务的技术。GCD存在于OS X v10.6及以后，以及iOS4.0及以后。
- 输入源【input source】。线程的一种异步事件源。输入源可以基于端口，也可以手动触发，输入源必须挂接【attached】到线程的runloop上。
- 可连接线程【joinable thread 】。一种终止时不立即释放资源【resources are not reclaimed】的线程。可连接线程必须显式分离【detached】或被其他线程连接，才可以释放资源【Joinable threads must be explicitly detached or be joined by another thread before the resources can be reclaimed】。可连接线程会向连接它的线程返回数据。
- library。UNIX系统监视底层系统事件的功能【feature】。更多信息请参考kqueue的man page。
- Mach端口派发源【Mach port dispatch source】。处理来自Mach端口的事件的派发源。
- 主线程【main thread】。一个特定的线程，进程创建时会创建该线程。主线程退出时，进程也随之结束。
- 互斥量【mutex】。一种锁，支持对共享资源的互斥访问。互斥锁同一时间只能被一个线程持有。尝试获取【acquire】其他线程持有的互斥锁，会导致当前线程阻塞，直到成功获取到锁。
- OpenCL【Open Computing Language】。在GPU上执行通用计算的标准技术。更多信息，请参考[OpenCL Programming Guide for Mac](https://developer.apple.com/library/content/documentation/Performance/Conceptual/OpenCL_MacProgGuide/Introduction/Introduction.html#//apple_ref/doc/uid/TP40008312)文档。
- operation【operation object】。NSOperation类的示例。operation可以将一个任务需要的代码和数据封装为一个可执行单元。
- 操作队列【operation queue】。NSOperationQueue类的示例。操作队列管理operation的执行。
- 私有派发队列【private dispatch queue】。你显示创建的派发队列，你要显式的retain和release。（译注：高级版本系统的ARC已经接管GCD的内存管理，版本号待标记）
- 程序【program】。代码和资源的结合体，可以运行，用来执行任务。程序不一定有图形用户界面，但图形应用也属于程序。
- 可重入【reentrant】。代码在一个线程执行时，它也可以在其他线程安全的启动执行。
- run loop。 一种事件处理循环，在循环中收到事件并派发到合适的处理器。
- runloop模式【run loop mode】。一种模式包含一组输入源、timer源以及runloop观察者的集合，有自己的名字。当runloop运行在某种模式下时，它只监视这个模式包含的源及观察者。
- runloop【run loop object】。NSRunLoop或CFRunLoopRef的实例，提供了在线程上运行事件处理循环的接口。
- runloop观察者【run loop observer】。runloop在执行的不同阶段会发出通知，观察者会接收这些通知。
- 信号量【semaphore】。一个控制共享资源访问的变量。互斥量和条件是不同类型的信号量。
- 信号【signal】。Unix中的一种使用进程外的信息操作进程的机制【A UNIX mechanism for manipulating a process from outside its domain】。系统使用信号来向应用传递重要信息。例如，是否应用执行了一条非法指令。更多信息，请参考signal的man page。
- 信号派发源【signal dispatch source】。处理Unix系统信号的派发源。当进程收到Unix信号时，派发源会调用你提供的事件处理器。
- 任务【task】。一定数量的要执行的工作。尽管有些技术中task有其他意义（尤其是Carbon Multiprocessing Services），但还是推荐将task定义为一种抽象概念，来指代一定数量的要执行的工作。
- 线程【thread】。进程中的一条执行路径。每个线程有自己的栈空间，但和进程的其他线程共享除此之外的其他内存空间。
- timer派发源【timer dispatch source】。处理定期事件的派发源。timer派发源周期性的调用你提供的事件处理器。


