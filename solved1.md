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
#### 问题，当出现某一个消息业务处理异常情况下，后续的消息无法消费处理，也无法补偿。

#### 问题定位过程

