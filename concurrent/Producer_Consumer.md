
Condition是在java 1.5中才出现的，它用来替代传统的Object的wait()、notify()实现线程间的协作，相比使用Object的wait()、notify()，使用Condition的await()、signal()这种方式实现线程间协作更加安全和高效。因此通常来说比较推荐使用Condition，阻塞队列实际上是使用了Condition来模拟线程间协作。

Condition是个接口，基本的方法就是await()和signal()方法；
Condition依赖于Lock接口，生成一个Condition的基本代码是lock.newCondition() 
 调用Condition的await()和signal()方法，都必须在lock保护之内，就是说必须在lock.lock()和lock.unlock之间才可以使用
　Conditon中的await()对应Object的wait()；

　Condition中的signal()对应Object的notify()；

  Condition中的signalAll()对应Object的notifyAll()。

Condition是一个多线程间协调通信的工具类，使得某个，或者某些线程一起等待某个条件（Condition）,只有当该条件具备( signal 或者 signalAll方法被带调用)时 ，这些等待线程才会被唤醒，从而重新争夺锁。

Condition实现生产者、消费者模式：

```java

public class ConTest {

    private int queueSize = 10;
    private PriorityQueue<Integer> queue = new PriorityQueue<>(queueSize);
    private Lock lock = new ReentrantLock();
    private Condition notFull = lock.newCondition();
    private Condition notEmpty = lock.newCondition();
    
    public static void main(String[] args) throws InterruptedException  {
        ConTest test = new ConTest();
        Producer producer = test.new Producer();
        Consumer consumer = test.new Consumer();
        producer.start();
        consumer.start();
        Thread.sleep(0);
        producer.interrupt();
        consumer.interrupt();
    }
    
    class Consumer extends Thread{
        @Override
        public void run() {
            consume();
        }
        volatile boolean flag=true;
        private void consume() {
            while(flag){
                lock.lock();
                try {
                    while(queue.isEmpty()){
                        try {
                            System.out.println("队列空，等待数据");
                            notEmpty.await();
                        } catch (InterruptedException e) {
                            flag =false;
                        }
                    }
                    queue.poll();                //每次移走队首元素
                    notFull.signal();
                    System.out.println("从队列取走一个元素，队列剩余"+queue.size()+"个元素");
                } finally{
                    lock.unlock();
                }
            }
        }
    }
    
    class Producer extends Thread{
        @Override
        public void run() {
            produce();
        }
        volatile boolean flag=true;
        private void produce() {
            while(flag){
                lock.lock();
                try {
                    while(queue.size() == queueSize){
                        try {
                            System.out.println("队列满，等待有空余空间");
                            notFull.await();
                        } catch (InterruptedException e) {

                            flag =false;
                        }
                    }
                    queue.offer(1);        //每次插入一个元素
                    notEmpty.signal();
                    System.out.println("向队列取中插入一个元素，队列剩余空间："+(queueSize-queue.size()));
                } finally{
                    lock.unlock();
                }
            }
        }
    }
}

```
