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
```java
