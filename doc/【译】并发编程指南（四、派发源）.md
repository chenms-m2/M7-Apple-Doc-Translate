[Concurrency Programming Guide 官档传送门](https://developer.apple.com/library/content/documentation/General/Conceptual/ConcurrencyProgrammingGuide/Introduction/Introduction.html#//apple_ref/doc/uid/TP40008091)

本文翻译自2012-12-13版的官档。
您可以在官档结尾的Document Revision History中查阅版本。
注：【】包含的部分引自原文。

## 派发源【Dispatch Sources】
和底层系统交互时，你的任务可能会消耗大量时间。产生内核或其他系统层调用会导致上下文切换，相比于在自己进程内处理工作，这更加耗时。因此，很多系统库提供了异步接口，你可以通过异步接口向系统提交请求，系统异步处理请求时，你可以先继续处理其他工作。GCD也是基于这样的机制，你可以使用block和派发队列来提交请求，并在任务完成后获得通知。

### 关于派发源
派发源是一种基础【fundamental】数据类型，用来协调处理特定的底层系统事件。GCD支持下述类型的派发源：
- timer派发源，产生周期性的通知。
- 信号【Signal】派发源，通知你收到Unix信号。
- 描述符【Descriptor】派发源，通知你文件、socket相关的操作，例如：  
    - 数据可被读取
    - 数据可写
    - 文件被删除、移动或重命名
    - 文件元信息修改
- 进程派发源，通知进程相关的事件，例如
    - 进程退出
    - 进程遇到fork或exec调用
    - 收到给本进程的信号【signal】
- Mach端口派发源，通知有Mach相关事件
- 自定义派发源，你可以自定义自触发

派发源取代了处理系统相关事件的回调函数。配置派发源时，你要指定想要监听的事件、收到通知后执行任务的队列。你可以指定处理事件的block或函数，当收到事件时，派发源会将block或函数作为任务提交到目标队列中执行。

和你手动提交任务到队列不同，派发源会提供一个连续的事件源。派发源会始终挂接【attached】在派发队列上，除非你手动取消它。挂接到派发队列上之后，每当派发源收到相关事件，就会将相关任务提交到队列中。有些事件会定期产生，如timer事件，不过大多数事件都是只在特定条件下产生。因此派发源会retain队列，以避免事件产生前队列就被释放。

为了避免派发队列中积累事件，派发源实现了事件合并【coalescing】机制。如果旧事件尚未处理的时候，就接到了新事件，派发源会将两者的数据合并。合并可能用替换掉旧数据，或更新数据，具体根据事件类型而定。例如，一个基于信号的派发源只提供最新的信号信息，以及上次处理后又接到多少信号。

### 创建派发源
创建派发源包括创建事件源【the source of the events】和派发源本身。事件源是处理事件必须的原生【native】数据结构。例如，基于描述符的派发源，你需要打开描述符，基于进程的源，你需要获取目标进程的进程ID。如果你已经有了事件源，你可以按下述步骤创建相关的派发源：
1. 使用dispatch_source_create创建派发源。
2. 配置派发源：
    - 为派发源指定事件处理器；请参考后文“编写及安装事件处理器”一节。
    - 对于timer源，使用dispatch_source_set_timer设置timer信息，参考后文的“创建timer”一节。
3. （可选）为派发源指定取消处理器；参考后文“安装取消处理器”一节。
4. 调用dispatch_resume 函数来开始处理事件；参考后文“挂起和回复派发源”一节。

由于派发源在使用前通常需要配置，因此dispatch_source_create返回的派发源默认是挂起状态。挂起的时候，派发源会收到事件，但不会去处理，这就给了你安装事件处理器及进行其他配置的机会。

接下来的小节展示了如何配置派发源。更详细的如何配置特定类型派发源的信息，请参考“派发源示例”章节。更多关于创建和配置派发源的信息，请参考Grand Central Dispatch (GCD) Reference文档（译注：原链接已失效）。

#### 编写及安装事件处理器【Writing and Installing an Event Handler】
你需要编写事件处理器来处理派发源产生的事件。事件处理器是个函数或block，通过调用dispatch_source_set_event_handler或dispatch_source_set_event_handler_f来将其安装到派发源。接收到事件时，派发源会将事件处理器作为任务提交到绑定的派发队列。

收到事件时，事件处理器负责处理。如果新事件到达时，你的事件处理器已经加入队列等待处理，那么派发源会合并两个事件。事件处理器通常只使用最新事件的信息，但对于某些类型的时间，也可能从其他事件或合并事件中获取信息。如果收到新事件时，事件处理器已经开始执行，那么派发源会暂存这些事件，直到事件处理器执行完毕。此时，派发源将新事件提交处理。

基于函数的事件处理器有一个上下文指针参数，该参数包含了派发源对象，没有返回值。基于block的事件处理器没有参数和返回值。
```
// Block-based event handler
void (^dispatch_block_t)(void)
 
// Function-based event handler
void (*dispatch_function_t)(void *)
```

在事件处理器中，你可以从派发源获取事件信息。基于函数的处理器会将派发源作为参数传入，基于block的处理器需要自己捕获，你可以在block中引用派发源变量。例如，下面的代码片段中，block捕获了source变量：
```
dispatch_source_t source = dispatch_source_create(DISPATCH_SOURCE_TYPE_READ,
                                 myDescriptor, 0, myQueue);
dispatch_source_set_event_handler(source, ^{
   // Get some data from the source variable, which is captured
   // from the parent context.
   size_t estimated = dispatch_source_get_data(source);
 
   // Continue reading the descriptor...
});
dispatch_resume(source);
```
在block中捕获变量通常更加灵活和动态。当然，block捕获的变量默认是只读的。尽管在特定的场景下，block可以修改捕获的变量，但你不应该在派发源绑定的事件处理器中这么做。派发源总是异步处理事件，因此当你处理事件时，捕获的变量的定义域可能已经不存在了。更多关于在block中如何捕获及使用变量的信息，请参考[Blocks Programming Topics](https://developer.apple.com/library/content/documentation/Cocoa/Conceptual/Blocks/Articles/00_Introduction.html#//apple_ref/doc/uid/TP40007502)

表4-1列举了事件处理器获取事件信息时可以使用的函数

表4-1 从派发源获取数据
- dispatch_source_get_handle
本函数返回派发源管理的底层数据类型。对于描述符派发源，本函数返回int型数据，其包含派发源关联的描述符。对于信号派发源，本函数返回int型数据，包含最新信号编号【the signal number】。对于进程派发源，本函数返回所监视的进程的pid_t类型的数据。对于Mach端口派发源，本函数返回mach_port_t类型的数据。对于其他类型的派发源，本函数的返回值为未定义。
- dispatch_source_get_data
本函数返回事件关联的未处理数据【pending data】。对于从文件读取数据的描述符派发源，本函数返回可供读取的字节数。对于向文件写入数据的描述符派发源，如果有空间可写入，本函数返回一个正整数值。对于监视文件系统活动的描述符派发源，返回能指示事件类型的常量，定义在dispatch_source_vnode_flags_t中。对于进程派发源，返回能指示事件类型的常量，定义在dispatch_source_proc_flags_t中。对于Mach端口派发源也是类似的，定义在dispatch_source_machport_flags_t中。对于自定义派发源，返回一个新的数据，新数据是由dispatch_source_merge_data函数合并已存在的数据和新数据得到的【this function returns the new data value created from the existing data and the new data passed to the dispatch_source_merge_data function】。
- dispatch_source_get_mask
本函数返回用于创建派发源的事件flags。对于进程派发源，函数返回派发源收到的事件掩码【mask】，定义在dispatch_source_proc_flags_t中。对于Mach端口派发源是类似的，定义在dispatch_source_mach_send_flags_t中。对于自定义OR派发源【custom OR dispatch source】，本函数返回用于合并数据的掩码。

编写及安装事件处理器的示例代码，请参考后文“派发源示例”一节。

#### 安装取消处理器【Cancellation Handler】
派发源被销毁时可以使用取消处理器来清理资源。对于大多数派发源来说，取消处理器是可选的，只有你的自定义行为和派发源绑定时才需要使用。对于描述符或Mach端口派发源来说，你必须提供取消处理器来关闭描述符或释放Mach端口，否则，当程序或系统其他部分复用这些资源时，可能会有潜在的bug。

通常在创建派发源的时候为其安装取消处理器。使用dispatch_source_set_cancel_handler或dispatch_source_set_cancel_handler_f函数，取决于你使用block还是函数实现。下面的示例代码展示了简单的取消处理器，它关闭了派发源打开的描述符。被捕获的fd变量包含描述符。
```
dispatch_source_set_cancel_handler(mySource, ^{
   close(fd); // Close a file descriptor opened earlier.
});
```
更完整的代码请参考后文的“从描述符读取数据”一节。

#### 改变目标队列
通常在创建派发源时，就为其指定绑定的派发队列（运行事件处理器及取消处理器的队列），你也可以随时使用dispatch_set_target_queue修改绑定的队列。例如，你可能想通过这种方式改变事件处理的优先级。

改变派发源的队列是个异步操作，派发源会尽早地完成改变。如果事件处理器已经被提交到队列，它依然会在之前的队列执行，在改变之后提交的，则会在新队列执行。

#### 派发源绑定的自定义数据
和GCD中很多数据类型一样，你可以使用dispatch_set_context为派发源关联自定义数据。你可以使用上下文指针存储事件处理器要使用的任意数据。如果你使用上下文指针存储了数据，你也应该为派发源添加取消处理器来清理不再使用的资源。

如果你使用block实现事件处理器，你可以在block中直接捕获变量来使用。尽管这可以减少使用上下文指针存储数据，但你依然要谨慎使用。派发源通常是长期生存的，你需要小心地对待捕获的指针。如果指针指向的对象随时可能被释放，那你需要复制或retain这个对象。此外，还要实现取消处理器来释放这个对象。

#### 派发源的内存管理
和其他GCD对象一样，派发源也采用引用计数机制。派发源初始计数为1，使用dispatch_retain和dispatch_release来增、减计数。当计数为0时，系统会自动销毁派发源。

派发源的拥有关系可以被外部管理，也可以内部管理。对于外部管理，其他对象或代码可以持有派发源，同时也有责任在不需要时release派发源。对于内部管理，派发源持有自己，需要再适当的时机release自己。外部管理是常见的方式，但某些场景下也可以进行内部管理，通常是你想要创建能自我管理的派发源。例如，一个派发源被设计为处理一个全局事件，你可能让它处理完该事件后立即退出。

### 派发源示例
接下来的几节展示了如何创建及配置常见的派发源。更多配置派发源的信息，请参考Grand Central Dispatch (GCD) Reference文档（译注：链接已失效）。

#### 创建Timer
timer派发源产生周期性的基于时间的事件。你可以使用它来执行周期性的任务。例如，游戏或其他图形相关的应用可能使用timer来更新屏幕和动画。你也可以使用timer去检查服务器上的信息是否更新。

所有timer派发源都是间隔timer【interval timers】，也就是说，一旦创建，它们会根据你指定的间隔周期性的获取事件。创建timer时，你必须指定允许误差【leeway】来帮助系统确定timer事件的时间精度。指定允许误差会让系统更灵活的管理电量及唤醒内核【cores】。例如，系统可能根据允许误差来调整timer的触发时间，来更好的和其他系统事件同步【align】。因此，只要可能，就给timer指定允许误差。

> 注意：即使你将允许误差指定为0，timer也不会精确到你想要的纳秒。系统会尽量满足你的需求，但不保证精确的触发时间。

计算机进入休眠时，timer派发源也会挂起，计算机唤醒后，这些timer派发源也会自动被唤醒。不同的配置会影响timer下次触发的时间。如果你使用dispatch_time函数或DISPATCH_TIME_NOW常量来设置派发源，timer会使用默认系统时钟来决定触发时间，但是，系统在休眠时，系统时钟不会前进。反之，如果你使用dispatch_walltime来设置派发源，timer派发源会根据挂钟时间【wall clock timer】来确定触发时间。使用挂钟时间的方式比较适合触发周期较长的timer，可以避免事件时间有太大偏差。

代码清单4-1展示了一个timer，它的周期是30秒，允许误差是1秒。由于周期较长，因此创建时使用了dispatch_walltime函数。第一个事件立即触发，随后的事件每30秒触发一次。示例中的MyPeriodicTask和MyStoreTimer代表你自定义的处理函数。

代码清单4-1 创建timer派发源
```
dispatch_source_t CreateDispatchTimer(uint64_t interval,
              uint64_t leeway,
              dispatch_queue_t queue,
              dispatch_block_t block)
{
   dispatch_source_t timer = dispatch_source_create(DISPATCH_SOURCE_TYPE_TIMER,
                                                     0, 0, queue);
   if (timer)
   {
      dispatch_source_set_timer(timer, dispatch_walltime(NULL, 0), interval, leeway);
      dispatch_source_set_event_handler(timer, block);
      dispatch_resume(timer);
   }
   return timer;
}
 
void MyCreateTimer()
{
   dispatch_source_t aTimer = CreateDispatchTimer(30ull * NSEC_PER_SEC,
                               1ull * NSEC_PER_SEC,
                               dispatch_get_main_queue(),
                               ^{ MyPeriodicTask(); });
 
   // Store it somewhere for later use.
    if (aTimer)
    {
        MyStoreTimer(aTimer);
    }
}
```
如果你想获取基于时间的事件，创建timer派发源是最常用的方式，但也有其他方式可以用。如果你想在指定时间之后执行任务，可以使用dispatch_after或dispatch_after_f函数。这两个函数的行为类似dispatch_async，区别是允许你指定提交block入队列的时间。这个时间参数可以是相对时间，也可以是绝对时间。

#### 从描述符【Descriptor】中读取数据
从文件或socket中读取数据，你需要【must】（译注：此处为什么是must）打开文件或socket，并创建DISPATCH_SOURCE_TYPE_READ类型的派发源。你指定的事件处理器应该有能力读取及处理文件描述符的内容，如果是从文件读取，它应该能将读到数据汇总，并生成合适的数据结构，如果是从socket读取，它应该能处理收到的网络数据。

读取数据时，你应该总是将描述符配置为非阻塞的。尽管你可以使用dispatch_source_get_data来查看可读取的数据大小，实际要读取时，数据大小可能已经变了。如果底层文件被删除了部分数据，或网络出现了错误，那么读取描述符可能会阻塞线程，你的事件处理器会卡在执行过程中。如果是串行队列，就造成死锁，如果是并发队列，会影响并发数。

代码清单4-2展示了如何使用派发源从文件读数据。示例中，事件处理器将文件中全部数据读取到缓冲区里，并且调用自定义函数来处理这些数据（调用方会在数据读取完毕后取消掉派发源）。为了保证没有数据时队列不会被阻塞，示例使用fcntl函数来将文件描述符配置为非阻塞的。取消处理器确保了数据读取完毕后，文件描述符可以被关闭。

代码清单4-2 从文件读取数据
```
dispatch_source_t ProcessContentsOfFile(const char* filename)
{
   // Prepare the file for reading.
   int fd = open(filename, O_RDONLY);
   if (fd == -1)
      return NULL;
   fcntl(fd, F_SETFL, O_NONBLOCK);  // Avoid blocking the read operation
 
   dispatch_queue_t queue = dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0);
   dispatch_source_t readSource = dispatch_source_create(DISPATCH_SOURCE_TYPE_READ,
                                   fd, 0, queue);
   if (!readSource)
   {
      close(fd);
      return NULL;
   }
 
   // Install the event handler
   dispatch_source_set_event_handler(readSource, ^{
      size_t estimated = dispatch_source_get_data(readSource) + 1;
      // Read the data into a text buffer.
      char* buffer = (char*)malloc(estimated);
      if (buffer)
      {
         ssize_t actual = read(fd, buffer, (estimated));
         Boolean done = MyProcessFileData(buffer, actual);  // Process the data.
 
         // Release the buffer when done.
         free(buffer);
 
         // If there is no more data, cancel the source.
         if (done)
            dispatch_source_cancel(readSource);
      }
    });
 
   // Install the cancellation handler
   dispatch_source_set_cancel_handler(readSource, ^{close(fd);});
 
   // Start reading the file.
   dispatch_resume(readSource);
   return readSource;
}
```
上面的示例中，MyProcessFileData函数会决定何时读取完毕，以及派发源何时可以被取消。默认的，派发源会重复的将事件处理器提交到队列中，知道数据读取完毕。如果socket连接关闭或文件读取完毕，派发源会自动的停止提交事件处理器。如果你确认不需要派发源了，可以直接取消它。

#### 向描述符写入数据
向文件或socket写数据和读数据是类似的。配置完描述符后，创建DISPATCH_SOURCE_TYPE_WRITE类型的派发源。创建完派发源后，系统会调用你的事件处理器来写数据。写数据完成后，调用dispatch_source_cancel来取消派发源。

和前文“从描述符中读取数据”一节中读取数据一样，你需要将描述符设置为非阻塞的。
【Whenever writing data, you should always configure your file descriptor to use non-blocking operations. Although you can use the dispatch_source_get_data function to see how much space is available for writing, the value returned by that function is advisory only and could change between the time you make the call and the time you actually write the data. If an error occurs, writing data to a blocking file descriptor could stall your event handler in mid execution and prevent the dispatch queue from dispatching other tasks. For a serial queue, this could deadlock your queue, and even for a concurrent queue this reduces the number of new tasks that can be started.】

代码清单4-3展示了如何使用派发源向文件写数据。创建完文件后，函数将文件描述符传给事件处理器。MyGetData函数提供了要写入的数据，这是你自定义的函数，你可以在里面实现创建数据的代码。数据写完后，事件处理器取消派发源，以防止它再次被调用，随后，派发源的拥有者release它。

代码清单4-3 向文件写数据
```
dispatch_source_t WriteDataToFile(const char* filename)
{
    int fd = open(filename, O_WRONLY | O_CREAT | O_TRUNC,
                      (S_IRUSR | S_IWUSR | S_ISUID | S_ISGID));
    if (fd == -1)
        return NULL;
    fcntl(fd, F_SETFL); // Block during the write. // 译注：此处和前文不符？前文是说设置为非阻塞的【Whenever writing data, you should always configure your file descriptor to use non-blocking operations】
 
    dispatch_queue_t queue = dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0);
    dispatch_source_t writeSource = dispatch_source_create(DISPATCH_SOURCE_TYPE_WRITE,
                            fd, 0, queue);
    if (!writeSource)
    {
        close(fd);
        return NULL;
    }
 
    dispatch_source_set_event_handler(writeSource, ^{
        size_t bufferSize = MyGetDataSize();
        void* buffer = malloc(bufferSize);
 
        size_t actual = MyGetData(buffer, bufferSize);
        write(fd, buffer, actual);
 
        free(buffer);
 
        // Cancel and release the dispatch source when done.
        dispatch_source_cancel(writeSource);
    });
 
    dispatch_source_set_cancel_handler(writeSource, ^{close(fd);});
    dispatch_resume(writeSource);
    return (writeSource);
}
```

#### 监视文件系统对象
如果你想监视一个文件系统对象的变化，可以创建一个DISPATCH_SOURCE_TYPE_VNODE类型的派发源。通过这个派发源，你可以在文件被删除、写入或重命名时获得通知。你也可以使用它来监视文件特定元信息的变化（如文件或大小或链接数量变化）。

> 注意：当派发源处理事件时，指定给它的文件描述符要保持打开状态。

代码列表4-4展示了在文件名改变时执行自定义操作（本例中你可以替换掉MyUpdateFileName的实现）。由于描述符是为这个派发源打开的，派发源需要包含取消处理器来关闭描述符。由于本例创建的文件描述符是和底层文件系统对象绑定的，因此派发源可以探测任意次数的文件名变化【detect any number of filename changes】。

代码列表4-4 观察文件名变化
```
dispatch_source_t MonitorNameChangesToFile(const char* filename)
{
   int fd = open(filename, O_EVTONLY);
   if (fd == -1)
      return NULL;
 
   dispatch_queue_t queue = dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0);
   dispatch_source_t source = dispatch_source_create(DISPATCH_SOURCE_TYPE_VNODE,
                fd, DISPATCH_VNODE_RENAME, queue);
   if (source)
   {
      // Copy the filename for later use.
      int length = strlen(filename);
      char* newString = (char*)malloc(length + 1);
      newString = strcpy(newString, filename);
      dispatch_set_context(source, newString);
 
      // Install the event handler to process the name change
      dispatch_source_set_event_handler(source, ^{
            const char*  oldFilename = (char*)dispatch_get_context(source);
            MyUpdateFileName(oldFilename, fd);
      });
 
      // Install a cancellation handler to free the descriptor
      // and the stored string.
      dispatch_source_set_cancel_handler(source, ^{
          char* fileStr = (char*)dispatch_get_context(source);
          free(fileStr);
          close(fd);
      });
 
      // Start processing events.
      dispatch_resume(source);
   }
   else
      close(fd);
 
   return source;
}
```

#### 监视信号通知【Monitoring Signals】
不熟悉信号，原文附上：
Monitoring Signals
UNIX signals allow the manipulation of an application from outside of its domain. An application can receive many different types of signals ranging from unrecoverable errors (such as illegal instructions) to notifications about important information (such as when a child process exits). Traditionally, applications use the sigaction function to install a signal handler function, which processes signals synchronously as soon as they arrive. If you just want to be notified of a signal’s arrival and do not actually want to handle the signal, you can use a signal dispatch source to process the signals asynchronously.

Signal dispatch sources are not a replacement for the synchronous signal handlers you install using the sigaction function. Synchronous signal handlers can actually catch a signal and prevent it from terminating your application. Signal dispatch sources allow you to monitor only the arrival of the signal. In addition, you cannot use signal dispatch sources to retrieve all types of signals. Specifically, you cannot use them to monitor the SIGILL, SIGBUS, and SIGSEGV signals.

Because signal dispatch sources are executed asynchronously on a dispatch queue, they do not suffer from some of the same limitations as synchronous signal handlers. For example, there are no restrictions on the functions you can call from your signal dispatch source’s event handler. The tradeoff for this increased flexibility is the fact that there may be some increased latency between the time a signal arrives and the time your dispatch source’s event handler is called.

Listing 4-5 shows how you configure a signal dispatch source to handle the SIGHUP signal. The event handler for the dispatch source calls the MyProcessSIGHUP function, which you would replace in your application with code to process the signal.

Listing 4-5  Installing a block to monitor signals
```
void InstallSignalHandler()
{
   // Make sure the signal does not terminate the application.
   signal(SIGHUP, SIG_IGN);
 
   dispatch_queue_t queue = dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0);
   dispatch_source_t source = dispatch_source_create(DISPATCH_SOURCE_TYPE_SIGNAL, SIGHUP, 0, queue);
 
   if (source)
   {
      dispatch_source_set_event_handler(source, ^{
         MyProcessSIGHUP();
      });
 
      // Start processing signals
      dispatch_resume(source);
   }
}
```
If you are developing code for a custom framework, an advantage of using signal dispatch sources is that your code can monitor signals independent of any applications linked to it. Signal dispatch sources do not interfere with other dispatch sources or any synchronous signal handlers the application might have installed.

For more information about implementing synchronous signal handlers, and for a list of signal names, see signal man page.

#### 监视进程
进程派发源可以监视特定进程的行为，并进行恰当的处理。父进程可能会使用这种派发源监视它创建的子进程。例如，父进程可以使用它来监视子进程的死亡。类似的，子进程可以使用它来监视父进程，以便于在父进程退出时也退出。

代码清单4-6展示了如何创建派发源来监视父进程的终结。本例中，父进程死亡后，派发源会设置内部状态让子进程意识到自己应该退出（你的程序可以替换本例中MySetAppExitFlag函数的实现，来设置恰当的终结flag）。由于派发源自主地运行，因此也拥有它自己，它可以在程序退出之前取消和释放自己。

代码清单4-6 监视父进程的死亡
```
void MonitorParentProcess()
{
   pid_t parentPID = getppid();
 
   dispatch_queue_t queue = dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0);
   dispatch_source_t source = dispatch_source_create(DISPATCH_SOURCE_TYPE_PROC,
                                                      parentPID, DISPATCH_PROC_EXIT, queue);
   if (source)
   {
      dispatch_source_set_event_handler(source, ^{
         MySetAppExitFlag();
         dispatch_source_cancel(source);
         dispatch_release(source);
      });
      dispatch_resume(source);
   }
}
```

### 取消派发源
除非你显式调用dispatch_source_cancel函数来取消派发源，否则它会一直存活。取消派发源会停止接收事件，并且无法撤销。因此，通常你取消派发源并立即释放它，如下：
```
void RemoveDispatchSource(dispatch_source_t mySource)
{
   dispatch_source_cancel(mySource);
   dispatch_release(mySource);
}
```
取消派发源是个异步操作。取消后，不会再处理新事件，但正在处理的事件会处理完毕，全部处理完后，派发源执行取消处理器（如果有的话）。

取消处理器是个释放内存和清理资源的好机会。如果派发源使用了描述符或Mach端口，你必须提供取消处理器来关闭描述符或销毁端口。其他类型的派发源则根据情况来决定是否提供取消处理器。例如，你在派发源的上下文指针中存储数据，就应该提供取消处理器来释放数据。更多信息，请参考前文的“安装取消处理器”一章。

### 挂起和恢复派发源
你可以dispatch_suspend和dispatch_resume来挂起或恢复派发源。这两个方法会增减派发源的挂起引用计数，因此你要注意两者的一一对应。
当你挂起派发源时，收到的事件会累积起来。【When you suspend a dispatch source, any events that occur while that dispatch source is suspended are accumulated until the queue is resumed. 】（译注：本句后半句使用了queue一次，后文也用到，但结合上下文看，挂起和恢复是指派发源）。派发源恢复后，会将累积的事件合并成一个事件处理。例如，你监视文件名的变化，合并后只保留最后一次文件名的变化。合并事件可以避免派发源恢复后顺便提交大量任务。
 