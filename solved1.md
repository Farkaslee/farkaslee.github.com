### 一次问题排查过程
#### 1.场景：在系统有收到并发消息，程序根据消息中的用户id 属性，会将消息分布到QUEUE中，然后会有线程池将消息取出异步消费的过程，在这个过程中为了保证可靠性，
会对未处理成功的消息进行JOB定时补偿。
```java
public class MessageStash {

    final int concurrentProcessors = Runtime.getRuntime().availableProcessors();

    private ExecutorService executorService;
    /**
     * 初始化queue
     */
    final Map<Integer,BlockingQueue<BaseMessageDTO>> stashQueue = new HashMap<>(Runtime.getRuntime().availableProcessors());

    /**
     * put message to blocking queue
     * @param message
     */
    public void publishEvent(BaseMessageDTO message){
    	Logger.info(this, "MessageStash.publishEvent - message %s", message.toString());
        //随机分布到对应的thread
    	Integer position = null;
    	if(message.getUserId()!=null) {
    		position  = (int) Math.abs(message.getUserId() % concurrentProcessors);
    	}else {
    		position  = Math.abs(message.hashCode() % concurrentProcessors);
    	}
        
        try {
        	Logger.info(this, "MessageStash.publishEvent - position %s", position);
            stashQueue.get(position).put(message);
        } catch (InterruptedException e) {
        	Logger.error(this, "no spache in stash queue, position:{},exception:{} ",position,e.getMessage());
        }
    }

    /**
     * 初始化blocking queue和线程池
     */
    public void init(MessageProcessorFactory messageProcessorFactory){
        ThreadFactory threadFactory = new ThreadFactoryBuilder().setNameFormat("message processor-%d").build();
        executorService = Executors.newFixedThreadPool(concurrentProcessors,threadFactory);
        for(int i=0;i<concurrentProcessors;i++){
            BlockingQueue<BaseMessageDTO> blockingQueue = Queues.newLinkedBlockingQueue();
            stashQueue.put(i,blockingQueue);
            executorService.execute(new ProcessorTask(blockingQueue,messageProcessorFactory));
            Logger.info(this, "created thread and blocking queue:{}"+i);
        }
    }

    public void destroy(){
        if (executorService!=null){
            executorService.shutdown();
        }
    }

    /**
     * 消息处理器,从blocking queue中获取消息，轮循处理。queue为空的是则等待。
     */
    public class ProcessorTask implements Runnable{

        private BlockingQueue<BaseMessageDTO> queue;
        private MessageProcessorFactory messageProcessorFactory;

        /**
         * 注入queue，messageTaskFactory
         * @param queue
         * @param messageProcessorFactory
         */
        public ProcessorTask(BlockingQueue<BaseMessageDTO> queue,MessageProcessorFactory messageProcessorFactory){
            this.queue = queue;
            this.messageProcessorFactory = messageProcessorFactory;
        }

        @Override
        public void run() {
            while (true){
                try {
                	Logger.debug(this, "ProcessorTask.run, start to take messageDTO from queue");
                	BaseMessageDTO messageDTO = queue.take();
                	Logger.debug(this, "ProcessorTask.run, take messageDTO from queue:%s", messageDTO.toString());
                	messageProcessorFactory.getProcessor(messageDTO).process(messageDTO);
                } catch (InterruptedException e) {
                    Logger.error("take in stash queue,exception:{} ", e.getMessage());
                }
            }
        }
    }
}
```
#### 问题：当出现某一个消息业务处理异常情况下，后续的消息无法消费处理，也无法补偿。

#### 问题定位过程
问题出现，怀疑几种情况：
1.因为线程池中的线程数是根据处理器个数设置，这里怀疑，处理器个数为1，当前处理的线程一直无法被调用到，后续的补偿也不能起作用
2.怀疑队列的问题

定位过程第一步找出来了部署环境中对应的应用进程 top 命令
通过jstack 360 查看当前线程的状况：
Full thread dump Java HotSpot(TM) 64-Bit Server VM (25.74-b02 mixed mode):

"Attach Listener" #111 daemon prio=9 os_prio=0 tid=0x00007f8efc005000 nid=0x308 waiting on condition [0x0000000000000000]
   java.lang.Thread.State: RUNNABLE

"message processor-11" #110 prio=5 os_prio=0 tid=0x00007f8e64003800 nid=0x2b5 waiting on condition [0x00007f8e75d9c000]
   java.lang.Thread.State: WAITING (parking)
        at sun.misc.Unsafe.park(Native Method)
        - parking to wait for  <0x00000000e3080470> (a java.util.concurrent.locks.AbstractQueuedSynchronizer$ConditionObject)
        at java.util.concurrent.locks.LockSupport.park(LockSupport.java:175)
        at java.util.concurrent.locks.AbstractQueuedSynchronizer$ConditionObject.await(AbstractQueuedSynchronizer.java:2039)
        at java.util.concurrent.LinkedBlockingQueue.take(LinkedBlockingQueue.java:442)
        at java.util.concurrent.ThreadPoolExecutor.getTask(ThreadPoolExecutor.java:1067)
        at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1127)
        at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:617)
        at java.lang.Thread.run(Thread.java:745)

"assetIncomeMessageRetryJob-3" #109 prio=5 os_prio=0 tid=0x00007f8ed8005800 nid=0x2a8 waiting on condition [0x00007f8e75c9b000]
   java.lang.Thread.State: TIMED_WAITING (parking)
        at sun.misc.Unsafe.park(Native Method)
        - parking to wait for  <0x00000000e2cb5af0> (a java.util.concurrent.locks.AbstractQueuedSynchronizer$ConditionObject)
        at java.util.concurrent.locks.LockSupport.parkNanos(LockSupport.java:215)
        at java.util.concurrent.locks.AbstractQueuedSynchronizer$ConditionObject.awaitNanos(AbstractQueuedSynchronizer.java:2078)
        at java.util.concurrent.ArrayBlockingQueue.poll(ArrayBlockingQueue.java:418)
        at java.util.concurrent.ThreadPoolExecutor.getTask(ThreadPoolExecutor.java:1066)
        at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1127)
        at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:617)
        at java.lang.Thread.run(Thread.java:745)

"assetSubjectChangedMessageRetryJob-4" #108 prio=5 os_prio=0 tid=0x00007f8ea8125000 nid=0x29b waiting on condition [0x00007f8e75f9e000]
   java.lang.Thread.State: TIMED_WAITING (parking)
        at sun.misc.Unsafe.park(Native Method)
        - parking to wait for  <0x00000000e33c3718> (a java.util.concurrent.locks.AbstractQueuedSynchronizer$ConditionObject)
        at java.util.concurrent.locks.LockSupport.parkNanos(LockSupport.java:215)
        at java.util.concurrent.locks.AbstractQueuedSynchronizer$ConditionObject.awaitNanos(AbstractQueuedSynchronizer.java:2078)
        at java.util.concurrent.ArrayBlockingQueue.poll(ArrayBlockingQueue.java:418)
        at java.util.concurrent.ThreadPoolExecutor.getTask(ThreadPoolExecutor.java:1066)
        at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1127)
        at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:617)
        at java.lang.Thread.run(Thread.java:745)

"message processor-10" #104 prio=5 os_prio=0 tid=0x00007f8e6c001800 nid=0x244 waiting on condition [0x00007f8e7609f000]
   java.lang.Thread.State: WAITING (parking)
        at sun.misc.Unsafe.park(Native Method)
        - parking to wait for  <0x00000000e3080470> (a java.util.concurrent.locks.AbstractQueuedSynchronizer$ConditionObject)
        at java.util.concurrent.locks.LockSupport.park(LockSupport.java:175)
        at java.util.concurrent.locks.AbstractQueuedSynchronizer$ConditionObject.await(AbstractQueuedSynchronizer.java:2039)
        at java.util.concurrent.LinkedBlockingQueue.take(LinkedBlockingQueue.java:442)
        at java.util.concurrent.ThreadPoolExecutor.getTask(ThreadPoolExecutor.java:1067)
        at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1127)
        at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:617)
        at java.lang.Thread.run(Thread.java:745)

"message processor-9" #100 prio=5 os_prio=0 tid=0x00007f8e70005800 nid=0x1f6 waiting on condition [0x00007f8e60fe7000]
   java.lang.Thread.State: WAITING (parking)
        at sun.misc.Unsafe.park(Native Method)
        - parking to wait for  <0x00000000e3080470> (a java.util.concurrent.locks.AbstractQueuedSynchronizer$ConditionObject)
        at java.util.concurrent.locks.LockSupport.park(LockSupport.java:175)
        at java.util.concurrent.locks.AbstractQueuedSynchronizer$ConditionObject.await(AbstractQueuedSynchronizer.java:2039)
        at java.util.concurrent.LinkedBlockingQueue.take(LinkedBlockingQueue.java:442)
        at java.util.concurrent.ThreadPoolExecutor.getTask(ThreadPoolExecutor.java:1067)
        at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1127)
        at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:617)
        at java.lang.Thread.run(Thread.java:745)

"message processor-8" #99 prio=5 os_prio=0 tid=0x00007f8e5c00e800 nid=0x1f5 waiting on condition [0x00007f8e761a0000]
   java.lang.Thread.State: WAITING (parking)
        at sun.misc.Unsafe.park(Native Method)
        - parking to wait for  <0x00000000e3080470> (a java.util.concurrent.locks.AbstractQueuedSynchronizer$ConditionObject)
        at java.util.concurrent.locks.LockSupport.park(LockSupport.java:175)
        at java.util.concurrent.locks.AbstractQueuedSynchronizer$ConditionObject.await(AbstractQueuedSynchronizer.java:2039)
        at java.util.concurrent.LinkedBlockingQueue.take(LinkedBlockingQueue.java:442)
        at java.util.concurrent.ThreadPoolExecutor.getTask(ThreadPoolExecutor.java:1067)
        at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1127)
        at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:617)
        at java.lang.Thread.run(Thread.java:745)

"message processor-7" #98 prio=5 os_prio=0 tid=0x0000000000f43800 nid=0x1f4 waiting on condition [0x00007f8e610e8000]
   java.lang.Thread.State: WAITING (parking)
        at sun.misc.Unsafe.park(Native Method)
        - parking to wait for  <0x00000000e3080470> (a java.util.concurrent.locks.AbstractQueuedSynchronizer$ConditionObject)
        at java.util.concurrent.locks.LockSupport.park(LockSupport.java:175)
        at java.util.concurrent.locks.AbstractQueuedSynchronizer$ConditionObject.await(AbstractQueuedSynchronizer.java:2039)
        at java.util.concurrent.LinkedBlockingQueue.take(LinkedBlockingQueue.java:442)
        at java.util.concurrent.ThreadPoolExecutor.getTask(ThreadPoolExecutor.java:1067)
        at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1127)
        at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:617)
        at java.lang.Thread.run(Thread.java:745)

"message processor-6" #96 prio=5 os_prio=0 tid=0x00007f8e68036000 nid=0x1e6 waiting on condition [0x00007f8e611e9000]
   java.lang.Thread.State: WAITING (parking)
        at sun.misc.Unsafe.park(Native Method)
        - parking to wait for  <0x00000000e3080470> (a java.util.concurrent.locks.AbstractQueuedSynchronizer$ConditionObject)
        at java.util.concurrent.locks.LockSupport.park(LockSupport.java:175)
        at java.util.concurrent.locks.AbstractQueuedSynchronizer$ConditionObject.await(AbstractQueuedSynchronizer.java:2039)
        at java.util.concurrent.LinkedBlockingQueue.take(LinkedBlockingQueue.java:442)
        at java.util.concurrent.ThreadPoolExecutor.getTask(ThreadPoolExecutor.java:1067)
        at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1127)
        at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:617)
        at java.lang.Thread.run(Thread.java:745)

"OracleTimeoutPollingThread" #95 daemon prio=10 os_prio=0 tid=0x00007f8ec85ea000 nid=0x1e5 waiting on condition [0x00007f8e614ea000]
   java.lang.Thread.State: TIMED_WAITING (sleeping)
        at java.lang.Thread.sleep(Native Method)
        at oracle.jdbc.driver.OracleTimeoutPollingThread.run(OracleTimeoutPollingThread.java:150)

"http-nio-0.0.0.0-8000-exec-25" #94 daemon prio=5 os_prio=0 tid=0x00007f8ea0011800 nid=0x1e4 waiting on condition [0x00007f8e617eb000]
   java.lang.Thread.State: WAITING (parking)
        at sun.misc.Unsafe.park(Native Method)
        - parking to wait for  <0x00000000e34b3be0> (a java.util.concurrent.locks.AbstractQueuedSynchronizer$ConditionObject)
        at java.util.concurrent.locks.LockSupport.park(LockSupport.java:175)
        at java.util.concurrent.locks.AbstractQueuedSynchronizer$ConditionObject.await(AbstractQueuedSynchronizer.java:2039)
        at java.util.concurrent.LinkedBlockingQueue.take(LinkedBlockingQueue.java:442)
        at org.apache.tomcat.util.threads.TaskQueue.take(TaskQueue.java:103)
        at org.apache.tomcat.util.threads.TaskQueue.take(TaskQueue.java:31)
        at java.util.concurrent.ThreadPoolExecutor.getTask(ThreadPoolExecutor.java:1067)
        at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1127)
        at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:617)
        at org.apache.tomcat.util.threads.TaskThread$WrappingRunnable.run(TaskThread.java:61)
        at java.lang.Thread.run(Thread.java:745)
其中 message processor 为自定义的线程池的名称。

发现所有的线程池都为waiting on condition，并没有别的应用线程。
#### 此时已经判断出部署环境的处理器个数为6，因为线程池的个数是6个。

后面导出了环境中的dump文件进行分析
jmap -dump:format=b,file=heap.bin 360
发现QUEUE中存在数据。

后续的代码review中
发现是线程中的异常处理只捕获了InterruptedException 未处理其他异常，这样在出现其他异常的情况下，线程会异常退出，
stashQueue 中的该线程对应的key的 BlockingQueue 会随线程消失。
```java
public class ProcessorTask implements Runnable{

        private BlockingQueue<BaseMessageDTO> queue;
        private MessageProcessorFactory messageProcessorFactory;

        /**
         * 注入queue，messageTaskFactory
         * @param queue
         * @param messageProcessorFactory
         */
        public ProcessorTask(BlockingQueue<BaseMessageDTO> queue,MessageProcessorFactory messageProcessorFactory){
            this.queue = queue;
            this.messageProcessorFactory = messageProcessorFactory;
        }

        @Override
        public void run() {
            while (true){
                try {
                	Logger.debug(this, "ProcessorTask.run, start to take messageDTO from queue");
                	BaseMessageDTO messageDTO = queue.take();
                	Logger.debug(this, "ProcessorTask.run, take messageDTO from queue:%s", messageDTO.toString());
                	messageProcessorFactory.getProcessor(messageDTO).process(messageDTO);
                } catch (InterruptedException e) {
                    Logger.error("take in stash queue,exception:{} ", e.getMessage());
                }
            }
        }
    }
```


### 线程的处理中一定要处理异常
