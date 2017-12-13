## 更好使用线程池

这篇文章结合Doug Lea大神在JDK1.5提供的JCU包，分别从线程池大小参数的设置、工作线程的创建、空闲线程的回收、阻塞队列的使用、任务拒绝策略、线程池Hook等方面来了解线程池的使用，其中涉及到一些细节包括不同参数、不同队列、不同拒绝策略的选择、产生的影响和行为、为更好的使用线程池奠定知识基础，其中值得注意的部分我用粗体标识。

ExecutorService基于池化的线程来执行用户提交的任务，通常可以简单的通过Executors提供的工厂方法来创建ThreadPoolExecutor实例。

线程池解决的两个问题：1）线程池通过减少每次做任务的时候产生的性能消耗来优化执行大量的异步任务的时候的系统性能。2）线程池还提供了限制和管理批量任务被执行的时候消耗的资源、线程的方法。另外ThreadPoolExecutor还提供了简单的统计功能，比如当前有多少任务被执行完了。

### 快速开始

为了使得线程池适合大量不同的应用上下文环境，ThreadPoolExecutor提供了很多可以配置的参数和可被用来扩展的钩子。然而，用户还可以通过使用Executors提供的一些工厂方法来快速创建ThreadPoolExecutor实例。比如：

使用Executors#newCachedThreadPool可以快速创建一个拥有自动回收线程功能且没有限制的线程池。

使用Executors#newFixedThreadPool可以用来创建一个固定线程大小的线程池。

使用Executors#newSingleThreadExecutor可以用来创建一个单线程的执行器。

如果上面的方法创建的实例不能满足我们的需求，我们可以自己通过参数来配置，实例化一个实例。

### 关于线程数大小参数设置需要知道的

ThreadPoolExecutor会根据corePoolSize和maximumPoolSize来动态调整线程池的大小:poolSize。

当任务通过executor提交给线程池的时候，我们需要知道下面几个点：

如果这个时候当前池子中的工作线程数小于corePoolSize，则新创建一个新的工作线程来执行这个任务，不管工作线程集合中有没有线程是处于空闲状态。

如果池子中有比corePoolSize大的但是比maximumPoolSize小的工作线程，任务会首先被尝试着放入队列，这里有两种情况需要单独说一下： 

a. 如果任务被成功的放入队列，则看看是否需要开启新的线程来执行任务，只有当当前工作线程数为0的时候才会创建新的线程，因为之前的线程有可能因为都处于空闲状态或因为工作结束而被移除。

b. 如果放入队列失败，则才会去创建新的工作线程。

如果corePoolSize和maximumPoolSize相同，则线程池的大小是固定的。

通过将maximumPoolSize设置为无限大，我们可以得到一个无上限的线程池。

除了通过构造参数设置这几个线程池参数之外我们还可以在运行时设置。

### 核心线程WarmUp

默认情况下，核心工作线程值在初始的时候被创建，当新任务来到的时候被启动，但是我们可以通过重写prestartCoreThread或prestartCoreThreads方法来改变这种行为。通常场景我们可以在应用启动的时候来WarmUp核心线程，从而达到任务过来能够立马执行的结果，使得初始任务处理的时间得到一定优化。

### 定制工作线程的创建

新的线程是通过ThreadFactory来创建的，如果没有指定，默认的Executors#defaultThreadFactory将被使用，这个时候创建的线程将都属于同一个线程组，拥有同样的优先级和daemon状态。扩展配置ThreadFactory，我们可以配置线程的名字、线程组合daemon状态。如果调用ThreadFactory#createThread的时候失败，将返回null，executor将不会执行任何任务。

### 空闲线程回收

如果当前池子中的工作线程数大于corePoolSize，如果超过这个数字的线程处于空闲的时间大于keepAliveTime，则这些线程将会被终止，这是一种减少不必要资源消耗的策略。这个参数可以在运行时被改变，我们同样可以将这种策略应用给核心线程，我们可以通过调用allowCoreThreadTimeout来实现。

### 选择合适的阻塞队列

所有的阻塞队列都可以被用来存放任务，但是使用不同的队列针对corePoolSize会表现不同的行为：

当池中工作线程数小于corePoolSize的时候，每次来任务的时候都会创建一个新的工作线程。

当池中工作线程数大于等于corePoolSize的时候，每次任务来的时候都会首先尝试将线程放入队列，而不是直接去创建线程。

如果放入队列失败，且当先池中线程数小于maximumPoolSize的时候，则会创建一个工作线程。

下面主要是不同队列策略表现:

直接递交：一种比较好的默认选择是使用SynchronousQueue，这种策略会将提交的任务直接传送给工作线程，而不持有。如果当前没有工作线程来处理，即任务放入队列失败，则根据线程池的实现，会引发新的工作线程创建，因此新提交的任务会被处理。这种策略在当提交的一批任务之间有依赖关系的时候避免了锁竞争消耗。值得一提的是，这种策略最好是配合unbounded线程数来使用，从而避免任务被拒绝。同时我们必须要考虑到一种场景，当任务到来的速度大于任务处理的速度，将会引起无限制的线程数不断的增加。

无界队列：使用无界队列如LinkedBlockingQueue没有指定最大容量的时候，将会引起当核心线程都在忙的时候，新的任务被放在队列上，因此，永远不会有大于corePoolSize的线程被创建，因此maximumPoolSize参数将失效。这种策略比较适合所有的任务都不相互依赖，独立执行。举个例子，如网页服务器中，每个线程独立处理请求。但是当任务处理速度小于任务进入速度的时候会引起队列的无限膨胀。

有界队列：有界队列如ArrayBlockingQueue帮助限制资源的消耗，但是不容易控制。队列长度和maximumPoolSize这两个值会相互影响，使用大的队列和小maximumPoolSize会减少CPU的使用、操作系统资源、上下文切换的消耗，但是会降低吞吐量，如果任务被频繁的阻塞如IO线程，系统其实可以调度更多的线程。使用小的队列通常需要大maximumPoolSize，从而使得CPU更忙一些，但是又会增加降低吞吐量的线程调度的消耗。总结一下是IO密集型可以考虑多些线程来平衡CPU的使用，CPU密集型可以考虑少些线程减少线程调度的消耗。

### 选择适合的拒绝策略

当新的任务到来的而线程池被关闭的时候，或线程数和队列已经达到上限的时候，我们需要去做一个决定，怎么拒绝这些任务。下面介绍一下常用的策略：

ThreadPoolExecutor#AbortPolicy：这个策略直接抛出RejectedExecutionException异常。

ThreadPoolExecutor#CallerRunsPolicy：这个策略将会使用Caller线程来执行这个任务，这是一种feedback策略，可以降低任务提交的速度。

ThreadPoolExecutor#DiscardPolicy：这个策略将会直接丢弃任务。

ThreadPoolExecutor#DiscardOldestPolicy：这个策略将会把任务队列头部的任务丢弃，然后重新尝试执行，如果还是失败则继续实施策略。

除了上面的几种策略，我们也可以通过实现RejectedExecutionHandler来实现自己的策略。

### 利用Hook嵌入你的行为

ThreadPoolExecutor提供了protected类型可以被覆盖的钩子方法，允许用户在任务执行之前会执行之后做一些事情。我们可以通过它来实现比如初始化ThreadLocal、收集统计信息、如记录日志等操作。这类Hook如beforeExecute和afterExecute。另外还有一个Hook可以用来在任务被执行完的时候让用户插入逻辑，如rerminated。

如果hook方法执行失败，则内部的工作线程的执行将会失败或被中断。

可访问的队列

getQueue方法可以用来访问queue队列以进行一些统计或者debug工作，我们不建议用作其他用途。同时remove方法和purge方法可以用来将任务从队列中移除。

关闭线程池

当线程池不在被引用并且工作线程数为0的时候，线程池将被终止。我们也可以调用shutdown来手动终止线程池。如果我们忘记调用shutdown，为了让线程资源被释放，我们还可以使用keepAliveTime和allowCoreThreadTimeOut来达到目的。

### 写在最后

JAVA本身提供的API已经可以让我们快速的进行基于线程池的多线程开发，但是我们必须要为我们写的代码负责，每一个参数的设置和策略的选择跟不同应用场景有绝对的关系。然而对于不同参数和不同策略的选择并不是一件容易的事情，我们必须要先回答一些基础问题：每创建一个线程，操作系统为我们做了哪些事情，这个线程的操作系统资源消耗主要在哪部分？假如我的应用场景是IO密集型的，那么我需要更多的线程还是更少的线程？假如我们的CPU操作和IO操作大概各占一半的话我们又需要如何选择？等等一些列问题。我认为、多线程开发是一件很容易的事情也是一件很不容易的事情。:)

### Contact

Having trouble with Pages? 

lfl969605@126.com