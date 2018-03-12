[Concurrency Programming Guide 官档传送门](https://developer.apple.com/library/content/documentation/General/Conceptual/ConcurrencyProgrammingGuide/Introduction/Introduction.html#//apple_ref/doc/uid/TP40008091)

本文翻译自2012-12-13版的官档。
您可以在官档结尾的Document Revision History中查阅版本。
注：【】包含的部分引自原文。

## 简介
并发【Concurrency】是指多件事情同时发生。随着多喝核CPU的普及，处理器的核会持续增长，开发者需要利用多核带来的优势。看上去OS X和iOS系统可以并行的运行多个程序，但多数程序运行在后台，不会连续占用处理器时间。只有在前台的程序吸引用户的注意，并占用大量的处理资源。如果一个应用的大部分工作只使用了CPU核的一部分，那么很多处理资源就被浪费了。

过去，程序中引入并发是通过创建新的线程。但是写多线程代码是个挑战，线程是偏底层的工具，你需要手动的维护它们。你需要根据系统和硬件的情况动态地决定线程的最佳数量，实现一个正确的多线程方案是很困难的。另外，使用多线程的同步机制，给软件设计引入了复杂度和风险，在提升性能方面也无法提供保障【 the synchronization mechanisms typically used with threads add complexity and risk to software designs without any guarantees of improved performance】。

OS X和iOS系统都提供新的处理异步任务的方案来取代传统的基于线程的方案。应用不再创建线程，而是定义任务，交给系统执行。通过让系统线程管理线程，应用获取了更好的扩展性，开发者的工作也得到简化，并且系统的方案提供了更好的性能。

本文档描述了应用中实现并发时要使用的技术，本文档描述的技术适用于OS X和iOS。

### 文档组织
本文档包含下面的章节：
- 并发与应用设计：介绍了基本的异步应用设计与技术，你可以异步地执行自定义的任务。
- 操作队列【Operation Queues】：介绍了如何通过OC对象来封装、执行任务。
- 派发队列【Dispatch Queues】：介绍了如何通过C的API来执行任务。
- 派发源【Dispatch Sources】：介绍了如何异步处理系统消息。
- 从多线程代码迁移：介绍了如何从基于线程的旧代码迁移到新技术。
文档还定义了相关的术语。

### 关于术语
进入并发的学习之前，有必要先定义一些术语，来避免混淆。熟悉Unix系统或老的OS X系统的开发者，会发现本文档中使用的“任务【task】”、“进程【process】”、“线程【thread】”术语和他们所知的有所不同。本文档按如下的描述使用这些术语：
- “线程”指代码的一条单独执行的路径。OS X中的线程基于POSIX thread API实现。
- “进程”指一个正在运行的可执行程序，它可以包含多条线程。
- “任务”是一个抽象概念，指代要执行的工作。
更多的术语请参考本文档的[Glossary](https://developer.apple.com/library/content/documentation/General/Conceptual/ConcurrencyProgrammingGuide/Glossary/Glossary.html#//apple_ref/doc/uid/TP40008091-CH104-SW2)一章。

### 更多参考
本文档主要关注并发编程的推荐技术，没有涉及到线程。如果你要使用线程，请参考[Threading Programming Guide](https://developer.apple.com/library/content/documentation/Cocoa/Conceptual/Multithreading/Introduction/Introduction.html#//apple_ref/doc/uid/10000057i).

## 并发与应用设计
早期，单位时间的最大运算速度取决于CPU的时钟速度。但随着处理器技术的进步，处理器对精密度【compact】、热特性【heat】及其他物理特性有了更高的要求，而这些条件制约了处理器时钟速度的进一步提升。因此芯片厂商努力寻找其它方式来提升芯片的处理能力。他们找到的方法是增加单个芯片上CPU核心的数量。通过增加核心数量，虽然CPU时钟速度与尺寸等其它因素没有改变，但单个芯片单位时间处理能力提升了。这个方案唯一的问题就是如何有效的利用多个核心。

为了利用多个核心，软件需要并行的执行任务。对于OS X、iOS这种现代多任务操作系统来说，是可以同时运行数百个程序的，将程序运行在不同的核心上是可能的。但大多是程序是系统的守护进程或后台进程，只会消耗很少的CPU资源，所以，利用多核的关键，是提升单个程序对多核的利用。

程序利用多核的传统方式是创建多个线程。然而，随着核心数目的增长，多线程也有一些问题。最大的问题就是多线程代码没法适应随意数量的CPU核心。你没法创建正好和匹配核心数的线程来让程序最高效。想达到高效，你需要知道当前可用的核心数（译注：不仅是硬件的核心数，更主要的是当前没被其他任务占用的核心数），而程序本心很难计算可用核心数。就算你可以计算出可用核心数，也会有其他问题，如何保持多线程的高效、如何处理多线程的交互等。

总结一下的话，程序需要一套方案来利用多核心。单个程序的多个工作处理也要能动态适应系统条件的变化（译注，如可用核心数的变化）。并且这套方案使用起来要简单，不应该为了适应多核引发大量的额外工作。好消息是苹果已经提供了这样的方案。本章我们就来看一下，这套方案涉及的技术，以及你如何调整设计来利用这套方案。

### 从多线程迁移【The Move Away from Threads】
尽管线程已经使用了很多年，而且还在被使用，但它很难动态的处理多任务。如果使用线程，你要扛起创建动态方案这一重担。你要根据系统条件的变化来动态维护合适数量的线程。除此之外，你的程序还要承担线程创建与管理的额外开销。

为了取代直接使用线程，OS X和iOS提供了新的异步设计方案来解决并发问题。异步函数在操作系统中已经存在了很多年，它们一般用于后台处理耗时任务，比如从硬盘中读取数据。异步函数被调用时，它会在后台启动异步任务，但不等任务执行完毕，就直接返回。通常，异步函数会获取后台线程，然后在该线程上启动任务，任务完成后会通知调用方（一般是通过回调函数）。过去，可能没有你想要的异步函数，你只能自己写该函数，并自己管理线程，不过现在OS X和iOS已经提供了相关的技术，让你直接执行异步任务，不必再自己管理线程。

其中的技术之一是GCD（Grand Central Dispatch）。这项技术将线程管理职责从你的程序转移到了系统。现在你所要做的仅仅是定义任务，然后添加到合适的派发队列中。GCD负责管理线程以及向线程注册任务。现在线程管理职责转移给了系统，那么GCD可以全局地进行任务管理与执行，这比传统的线程更高效。

操作队列技术【Operation queues】和GCD类似，但它的API是Objective-C（后文简称OC）级别的。你可以定义任务，并添加到操作队列，队列会负责任务的注册及执行。和GCD一样，操作队列也会为你管理线程，保证任务可以快速高效地执行。

接下来几节，更详细地描述了派发队列、操作队列以及其他可能会用的异步技术。

#### 派发队列【Dispatch Queues】
派发队列是基于C的任务执行机制。派发队列串行或并行地执行任务，但两者都是遵循先进先出【FIFO】的（换句话说，先加入队列的任务先执行）。串行队列一次只执行一个任务，前一个任务执行完毕后，后一个任务才出队列执行。而并行队列会在自己的能力范围内尽量多地启动任务，不会去管已启动的任务是否执行完毕。
派发队列还提供了其他好处：
- 提供了简单的接口
- 提供了自动全局的线程管理
- 提供了【speed of tuned assembly】
- 内存使用更加高效（线程不再占用程序内存）
- 不会让内核负荷过低【the kernel under load】
- 异步派发任务不会死锁
- 在抢占式机制下优雅地实现可伸缩性【scale gracefully under contention】 
- 串行队列比锁及同步原语更加有效

提交给派发队列的任务必须被封装为函数或block。block是OSX v10.6和iOS 4.0引入的功能，概念上类似函数指针，不过有它额外的好处。你可以在方法或函数中定义block，这样block可以访问方法或函数中的局部变量。如果你将block提交给派发队列，block会被自动复制到堆上。上面这些都方便用少量代码实现动态任务。

派发队列是GCD的一部分，也是C运行时的一部分。更详细的内容，请参考[Dispatch Queues](https://developer.apple.com/library/content/documentation/General/Conceptual/ConcurrencyProgrammingGuide/OperationQueues/OperationQueues.html#//apple_ref/doc/uid/TP40008091-CH102-SW1)一章，如果你想更多的了解block，请参考[Blocks Programming Topics](https://developer.apple.com/library/content/documentation/Cocoa/Conceptual/Blocks/Articles/00_Introduction.html#//apple_ref/doc/uid/TP40007502)文档.

#### 派发源【Dispatch Sources】
派发源也是基于C的，它用于异步处理特定类型的系统事件。事件发生时，派发源会封装该事件的信息，并向派发队列提交一个函数或block。你可以用派发源来监视下述的系统事件：
- 计时器
- 信号处理【Signal handlers】
- 描述符相关事件【Descriptor-related events】
- 进程相关事件
- Mach端口事件
- 你主动触发的自定义事件

派发源也是GCD的一部分，更多信息请参考[Dispatch Sources](https://developer.apple.com/library/content/documentation/General/Conceptual/ConcurrencyProgrammingGuide/GCDWorkQueues/GCDWorkQueues.html#//apple_ref/doc/uid/TP40008091-CH103-SW1)一章。

#### 操作队列【Operation Queues】
操作队列是Cocoa实现的类似于并发派发队列的机制，在Cocoa中体现为NSOperationQueue类。和派发队列的先进先出策略不同的是，操作队列中可以根据不同的因素决定任务执行的顺序，一个主要的因素是任务的依赖关系，依赖其他任务意味着等其他任务执行完毕才执行自身。设定任务的依赖关系可以创建复杂的执行顺序图。

提交给操作队列的任务必须是NSOperation类型的实例。NSOperation对象是OC对象，它封装了任务的相关逻辑与数据。NSOperation本质上可认为是个抽象类，因此你一般需要自定义其子类。当然Foundation中也提供了几个具体子类。

NSOperation对象可以生产KVO通知，因此你可以方便的监视任务执行状态。尽管操作队列是并发的，你也可以用依赖关系来构造串行逻辑。

更多信息请参考[Operation Queues](https://developer.apple.com/library/content/documentation/General/Conceptual/ConcurrencyProgrammingGuide/OperationObjects/OperationObjects.html#//apple_ref/doc/uid/TP40008091-CH101-SW1)一章。

### 异步设计技术
在你确认要引入并发之前，最好问问自己，是否真的需要并发。并发可以让你主线程专心的响应用户事件，来提供良好的用户体验。它也可以利用多核的处理能力，单位时间完成更多的工作。但是，它也会提高资源消耗【add overhead】，并且给代码引入额外的复杂度，使代码编写与调试都更加困难。

由于并发会增加复杂度，因此不适合在开发要结束的时候引入。如果要引入并发，你要仔细思考你要执行的任务及相关的数据结构。如果处理不当的话，你的任务可能会更慢，应用体验更差。因此，你应该在设计之初，就确定目标，并仔细思考达成目标的方式。

每个项目都有不同的需要和任务，不可能一篇文档就能讲清楚如何设计应用及其任务。然而，下面几节还是尝试提供一些建议，来指引您在设计时做出更好的选择。

#### 定义程序目标行为【Expected Behavior】
在考虑是否引入并发之前，先定义出你认为的程序的目标行为。理解了程序的目标可以验证你的设计，也能让你了解引入并发后预期的性能提升。

首先要做的是列举你的程序要执行的任务，以及任务相关的数据结构。首先，你可以列举出以用户事件（如点击按钮）为开端的任务。这个级别的任务有独立的业务逻辑【discrete behavior】，以及明确的开始点与结束点。然后，你可以列出用户事件无关的任务，如计时器任务。

完成这个粗粒度【high-level】的任务列表后，你可以进一步将每个任务分解为更小的步骤。在这个粒度，你可以重点关注对数据结构和对象的修改，以及这些修改如何影响应用的整体状态。你也需要注意对象及数据结构间的依赖关系。例如，如果一个任务要对数组中每个对象都做同样的处理，要关注对一个对象的处理是否会影响其他对象。如果对象的修改是互不影响的，此处就有可能引入并发。

#### 提取出可执行的业务逻辑单位【Factor Out Executable Units of Work】
理解了你的应用的任务，你应该能够确定在何处引入并发。如果改变每一步的顺序会影响结果，那么这些步骤应该串行执行。如果改变步骤顺序不影响结果，那么可以并发地执行这些步骤。不管哪种情况，你都要提取出可执行单位，来封装要执行的步骤。通常你会封装出block或操作对象【operation object】，并将其派发到合适的队列。

提取可执行单位时，不要担心单位过多，至少初期不用担心。尽管线程调度有开销，但派发队列或操作队列都做了优化，让开销比传统线程小。因此，及时你提取的单位粒度很小，使用队列也能获取比传统线程更好的性能。当然，你应该测试实际的性能，并按需要调整单位粒度。但初期，单位粒度小些没有问题。

#### 确认合适的队列
现在你已经将任务封装成了block或operation，接下来你需要选择执行任务的队列。这要根据任务的特定需求及执行顺序。

如果你将任务封装成了block，你可以将其加入串行或并行的派发队列。如果需要特定顺序就加入串行队列，繁殖，就加入并发队列，或加入不同的队列，取决于你的需求。

如果你将任务封装成了operation，那么你没得选择，只能加入并行的操作队列，但你可以配置operation的依赖关系，来模仿串行队列。

#### 提高性能的小提示
除了前文所说的方式，使用队列时还有几种方式可以提升整体性能：
- 如果内存使用是个制约点，任务中可以考虑直接使用计算值【Consider computing values directly within your task if memory usage is a factor】。
如果你的程序有内存访问瓶颈【memory bound】，直接使用计算值可能比从主存中加载缓存值更快。直接使用计算值会使用寄存器及CPU的高速缓存，要快于主存访问。当然，只有确认使用该方式真的提升了性能，才应该使用。
- 尽快鉴别出串行任务，并尝试使之支持并行。
任务必须串行通常是因为访问了共享资源，可以考虑移除共享资源。你可以考虑复制该资源或直接移除【eliminate】该资源。
- 避免使用锁。
使用派发队列和操作队列的话，多数情况下都不需要使用锁。如果要访问共享资源，可以考虑使用串行派发队列（或为operation添加依赖模拟串行）。
- 尽量使用系统库提供的API。使用并发的最好方式就是借助系统库内建的并发支持。很多系统库都基于线程或其他技术实现了并发。定义任务时，应该优先查看系统库是否已经提供了函数或方法可以让你并发执行任务。使用系统API可以减少你的工作量，并提供最大的并发性。

### 性能提示【Performance Implications】
派发队列、派发源以及操作队列可以让你更轻松的为程序引入并发，但它们并不能保证提高程序性能以及响应能力。你需要恰当地使用这些技术，以保证既满足你的需要，又不会给应用的其他资源带来不利影响。例如，你创建了10000个operation并提交到了操作队列中，这会导致你的应用开辟大量的内存，这可能导致换页【paging】，导致性能降低。

在为程序引入并发前（使用队列或线程），你应该收集一组能反应程序当前性能的数据作为基准数据。当你引入并发后，再收集一组数据和基准数据比较，以此评价引入并发后对性能造成的影响。如果引入并发后性能反而下降了，你可以使用工具检查可能的原因。

关于性能及其测量工具，请参考[Performance Overview](https://developer.apple.com/library/content/documentation/Performance/Conceptual/PerformanceOverview/Introduction/Introduction.html#//apple_ref/doc/uid/TP40001410)文档。

### 并发和其他技术
提取代码封装任务是提高并发的最好方式。但是这个方式未必适合所有场景。某些场景下，另一些技术可能是更好的选择。本节介绍了你可能使用到的其他技术。

#### OpenCL和并发
OSX中，OpenCL（Open Computing Language）是一种标准的在GPU上处理通用计算任务【general-purpose computations on a computer’s graphics processor】的技术。如果你有一组定义良好的计算集合，想要批量应用在大量数据上，OpenCL可能是一个合适的技术。例如，你可能会使用OpenCL来对图片上的所有像素做过滤处理，或者对很多数值执行浮躁的数学运算。换句话说，OpenCL更适合用在在数据可以并行【parallel】处理的问题集上。

尽管OpenCL擅长处理海量并行数据运算，但它不太适合更加通用的计算任务【more general-purpose calculations】。为了在GPU上运算，需要准备并传输大量的数据与必要的处理逻辑【required work kernel】给显卡。类似的，OpenCL又要取回大量的计算结构。因此，需要和系统交互的任务不适合交给OpenCL。例如，OpenCL不适合处理文件或网络流中的数据。相反，你应该让OpenCL来处理更加自包含【self-contained】的工作，这样当你传输给GPU后它就可以独立的处理。

更多信息请参考[OpenCL Programming Guide for Mac](https://developer.apple.com/library/content/documentation/Performance/Conceptual/OpenCL_MacProgGuide/Introduction/Introduction.html#//apple_ref/doc/uid/TP40008312)。

#### 什么时候使用线程
尽管我们非常推荐派发队列与操作队列来处理并发，但它们不是万能的。在极少数情况下，还是会用到线程。如果你要使用自定义的线程，你还是要努力减少创建的线程，而且确保只在其他方式处理不了的情况才使用线程。

线程在需要实时地执行代码【must run in real time】的场景还是很有用的。派发队列会尝试尽早地执行任务，但不保障实时性【they do not address real time constraints】。如果你希望后台执行的代码行为更可预期，线程是个不错的选择。

是否使用线程，还是要谨慎，在确定绝对必须的情况下才使用。更多信息请参考[Threading Programming Guide](https://developer.apple.com/library/content/documentation/Cocoa/Conceptual/Multithreading/Introduction/Introduction.html#//apple_ref/doc/uid/10000057i)文档。


