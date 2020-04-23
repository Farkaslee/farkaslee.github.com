### Semaphore

Semaphore也是一个线程同步的辅助类，可以维护当前访问自身的线程个数，并提供了同步机制。使用Semaphore可以控制同时访问资源的线程个数，例如，实现一个文件允许的并发访问数。

Semaphore的主要方法摘要：

　　void acquire():从此信号量获取一个许可，在提供一个许可前一直将线程阻塞，否则线程被中断。

　　void release():释放一个许可，将其返回给信号量。

　　int availablePermits():返回此信号量中当前可用的许可数。

　　boolean hasQueuedThreads():查询是否有线程正在等待获取。
```java
@Service
public class SequenceService {
	
	@Autowired
	private SequenceMapper sequenceMapper;
	
	private Integer THRESHOLD = 10;
	private ConcurrentLinkedDeque<Long> queue = new ConcurrentLinkedDeque<Long>();
	private Semaphore semaphore = new Semaphore(1);

	/**
	 * 批量获取code
	 * 
	 * @param size 批次大小
	 *            
	 * @return
	 */
	public List<Long> getCode(String sequenceType, int size) {
		List<Long> resultList = new ArrayList<Long>();
		for (int i = 0; i < size; i++) {
			Long element = queue.poll();
			if (element == null || queue.size() < THRESHOLD) {
				// 不能获取信号量，立即返回
				if (semaphore.tryAcquire()) {
					new Thread(new Runnable() {
						@Override
						public void run() {
							Logger.info(getClass(), String.format("%s - 获取到信号量，开始向queue中加载code", Thread.currentThread().getName()));
							load(sequenceType);
						}
					}).start();
				}
				// 缓存未命中，调SEQUENCE生成code
				if (element == null) {
					List<Long> list = getCodeFromExternal(sequenceType, 1);
					Logger.info(getClass(), String.format("%s - 从DB中直接获取 - %s", Thread.currentThread().getName(), list.get(0)));
					resultList.add(list.get(0));
					continue;
				}
			}
			resultList.add(element);
		}
		return resultList;
	}

	/**
	 * 缓存产品code
	 */
	public void load(String sequenceType) {
		Logger.info(getClass(), String.format("%s - 开始向queue中加载code", Thread.currentThread().getName()));
		try {
			if (queue.size() >= THRESHOLD) {
				return;
			}
			List<Long> codes = getCodeFromExternal(sequenceType, THRESHOLD * 2);
			queue.addAll(codes);
		} finally {
			Logger.info(getClass(), String.format("%s- queue size：%s", Thread.currentThread().getName(), queue.size()));
			semaphore.release();
		}

	}

	/**
	 * 从数据库加载
	 * 
	 * @param size
	 * @return
	 */
	private List<Long> getCodeFromExternal(String sequenceType, int size) {
		List<Long> codes=sequenceMapper.selectNextVal(sequenceType, size);
		return codes;
	}
}
```

Semaphore 示例
```java
public class SemaphoreTest {
    static class MyThread implements Runnable {

        private int value;
        private Semaphore semaphore;

        public MyThread(int value, Semaphore semaphore) {
            this.value = value;
            this.semaphore = semaphore;
        }

        @Override
        public void run() {
            try {
                semaphore.acquire(); // 获取permit
                System.out.println(String.format("当前线程是%d, 还剩%d个资源，还有%d个线程在等待",
                        value, semaphore.availablePermits(), semaphore.getQueueLength()));
                // 睡眠随机时间，打乱释放顺序
                Random random =new Random();
                Thread.sleep(random.nextInt(1000));
                semaphore.release(); // 释放permit
                System.out.println(String.format("线程%d释放了资源", value));
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
    }

    public static void main(String[] args) {
        Semaphore semaphore = new Semaphore(3);
        for (int i = 0; i < 10; i++) {
            new Thread(new MyThread(i, semaphore)).start();
        }
    }
}
```

### Support or Contact

Having trouble with Pages?

lfl969605@126.com
