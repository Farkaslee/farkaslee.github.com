在学习Java 多线程并发开发过程中，了解到util.concurrent.DelayQueue类的主要作用：是一个无界的BlockingQueue，用于放置实现了Delayed接口的对象，其中的对象只能在其到期时才能从队列中取走。这种队列是有序的，即队头对象的延迟到期时间最长。注意：不能将null元素放置到这种队列中。

jdk 1.8 DelayQueue，带有延迟元素的线程安全队列，当非阻塞从队列中获取元素时，返回最早达到延迟时间的元素，或空（没有元素达到延迟时间）。DelayQueue的泛型参数需要实现Delayed接口，Delayed接口继承了Comparable接口，DelayQueue内部使用非线程安全的优先队列（PriorityQueue），并使用Leader/Followers模式，最小化不必要的等待时间。DelayQueue不允许包含null元素。

Leader/Followers模式，借用其他博客的话来解释下：
打个比方：

话说一个地方有一群有组织无纪律的人从事山贼这个很有前途的职业。

一般就是有一个山贼在山路口察看，其他人在林子里面睡觉。

假如发现有落单的过往客商，望风的山贼就会弄醒一个睡觉的山贼，然后自己去打劫。

醒来的山贼接替作望风的事情。

打劫的山贼搞定以后，就会去睡觉，直到被其他望风的山贼叫醒来望风为止。

有时候过往客商太多，而山贼数量不够，有些客商就能侥幸平安通过山岭(所有山贼都去打劫其他客商了)。

下面是这个模式的计算机版本：

有若干个线程(一般组成线程池)用来处理大量的事件

有一个线程作为领导者，等待事件的发生；其他的线程作为追随者，仅仅是睡眠。

假如有事件需要处理，领导者会从追随者中指定一个新的领导者，自己去处理事件。

唤醒的追随者作为新的领导者等待事件的发生。

处理事件的线程处理完毕以后，就会成为追随者的一员，直到被唤醒成为领导者。

假如需要处理的事件太多，而线程数量不够(能够动态创建线程处理另当别论)，则有的事件可能会得不到处理。

### DelayQueue 项目使用实例
```java
@Service
public class DelayQueueService implements InitializingBean {

	@Autowired
	private MessageProcessorFactory messageProcessorFactory;
	@Autowired
	private MessageService messageService;
	@Autowired
	private ResolverRegistry resolverRegistry;
	
	private DelayQueue<DelayMessage<BaseMessageDTO>> delayQueue = new DelayQueue<>();

	private long DELAY_MS = 5000;

	private ExecutorService executorService;
	
	@Override
	public void afterPropertiesSet() throws Exception {
		Logger.info(this,"start to initialize DelayQueueService");
		ThreadFactory threadFactory = new ThreadFactoryBuilder().setNameFormat("delayQueue processor-%d").build();
        executorService = Executors.newFixedThreadPool(1,threadFactory);
        executorService.execute(new DelayQueueTask(messageProcessorFactory, delayQueue));
	}

	@PreDestroy
	public void preDestroy() {
		Logger.info(this,"start to shutdown executorService of DelayQueueService");
		executorService.shutdown();
	}
	
	public void put(DelayMessage<BaseMessageDTO> delayMessage) {
		if (delayQueue == null) {
			Logger.error(this, "DelayQueue must to be initilized");
			return;
		}
		if (delayQueue.size() < 1000) {
			delayQueue.put(delayMessage);
		}
	}

	public long getExpireTime() {
		return System.currentTimeMillis() + DELAY_MS;
	}

	class DelayQueueTask implements Runnable {

		private MessageProcessorFactory factory;

		private DelayQueue<DelayMessage<BaseMessageDTO>> queue;
		
		public DelayQueueTask(MessageProcessorFactory factory, DelayQueue<DelayMessage<BaseMessageDTO>> delayQueue) {
			this.factory= factory;
			this.queue=delayQueue;
		}
		
		@Override
		public void run() {
			while (true) {
				try {
					Logger.info(this, "DelayQueueTask.run - start to take messageDTO from queue");
					DelayMessage<BaseMessageDTO> delayMessage = queue.take();
					Logger.info(this, "DelayQueueTask.run - process messageDTO from delay queue:%s", delayMessage.toString());
					//为防止消息已经被处理，需要重新获取消息，再根据消息状态判断是否需要处理
					Message message=messageService.queryById(delayMessage.getMessage().getId(), delayMessage.getMessage().getPartitionKey());
					Logger.info(this, "DelayQueueTask.run - retrieve message from database:%s", message.toString());
					if(MessageStatusEnum.getByCode(message.getStatus()).isRetriable()) {
						Class<?> clazz = resolverRegistry.getPayloadType(TradeTypeEnum.valueOfCode(message.getTradeType()), ProductCategoryEnum.valueOfCode(message.getProdCategory()));
						BaseMessageDTO baseMessageDTO=messageService.getMessageDTO(message, clazz);
						factory.process(baseMessageDTO);
						Logger.info(this, "DelayQueueTask.run, message was successfully processed :%s", delayMessage.getMessage().toString());
					}else {
						Logger.info(this, "DelayQueueTask.run, message have been handled by other thread :%s", delayMessage.getMessage().toString());
					}
				} catch (Throwable e) {
					Logger.error("ProcessorTask.run - exception:{} ", e.getMessage());
				}
			}
		}
	}

}
```
```java


public class DelayMessage<T> implements Delayed {

	private T message;

	private long expireAt;

	public DelayMessage(T message, long expireAt) {
		this.message = message;
		this.expireAt = expireAt;
	}

	@Override
	public int compareTo(Delayed o) {
		return 0;
	}

	@Override
	public long getDelay(TimeUnit unit) {
		return unit.convert(this.expireAt - System.currentTimeMillis(), TimeUnit.MILLISECONDS);
	}

	public T getMessage() {
		return message;
	}

}
```
