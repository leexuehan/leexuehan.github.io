---
title: ThreadPool 之学习
tags: Concurrency
date: 2015-08-19 16:21:59
categories: 技术
---

## 什么是线程池

通常情况下，我们使用线程的时候就去new一个线程，这样实现起来非常简便，但是就会有一个问题：

在高并发且并发任务执行时间较短的情况下，频繁创建和销毁线程的成本太高，这涉及到一个线程

如果并发的线程数量很多，并且每个线程都是执行一个时间很短的任务就结束了，这样频繁创建线程就会大大降低系统的效率，因为频繁创建线程和销毁线程需要时间。由于Java线程依赖于操作系统内核线程，所以线程创建需要切换到**内核态**，创建和销毁的消耗比较大。 

另外线程属于稀缺资源，不能无限制的创建，通过线程池可以进行统一的创建和监控。 遇到了性能瓶颈，吞吐量上不去的时候，并不是一味的加大线程数就能提高性能，线程达到一定的程度，上下文切换会消耗很大的CPU，并且可能是在做无用功。

那么有没有一种办法使得线程可以复用，就是执行完一个任务，并不被销毁，而是可以继续执行其他的任务？

在Java中可以通过线程池来达到这样的效果。

### 使用线程池的好处？

1. 减少了创建和销毁线程的次数，每个工作线程都可以被重复利用，可执行多个任务。

2. 可以根据系统的承受能力，调整线程池中工作线线程的数目，防止因为消耗过多的内存，而把服务器累趴下(每个线程需要大约1MB内存，线程开的越多，消耗的内存也就越大，最后死机)。

### 线程池怎么使用？

Java通过Executors提供四种线程池，分别为：

- **newCachedThreadPool**     创建一个可缓存线程池，如果线程池长度超过处理需要，可灵活回收空闲线程（一般情况下如果一个线程有60s没有被使用，则回收之），若无可回收，则新建线程。对于线程池的长度并没有任何限制。
- **newFixedThreadPool**      创建一个定长线程池，可控制线程最大并发数，超出的线程会在队列中等待。
- **newScheduledThreadPool**  创建一个定长线程池，支持定时及周期性任务执行。
- **newSingleThreadExecutor** 创建一个单线程化的线程池，它只会用唯一的工作线程来执行任务，保证所有任务按照指定顺序(FIFO, LIFO, 优先级)执行。

#### 线程池的创建

创建一个线程池需要的参数：

**corePoolSize**（线程池的基本大小）：当提交一个任务到线程池时，线程池会创建一个线程来执行任务，即使其他空闲的基本线程能够执行新任务也会创建线程，**等到需要执行的任务数大于线程池基本大小时就不再创建**。如果调用了线程池的prestartAllCoreThreads方法，线程池会提前创建并启动所有基本线程。

**runnableTaskQueue**（任务队列）：用于保存等待执行的任务的阻塞队列。 可以选择以下几个阻塞队列。

- ArrayBlockingQueue：是一个基于数组结构的有界阻塞队列，此队列按 FIFO（先进先出）原则对元素进行排序。
- LinkedBlockingQueue：一个基于链表结构的阻塞队列，此队列按FIFO （先进先出） 排序元素，吞吐量通常要高于ArrayBlockingQueue。静态工厂方法Executors.newFixedThreadPool()使用了这个队列。
- SynchronousQueue：一个不存储元素的阻塞队列。每个插入操作必须等到另一个线程调用移除操作，否则插入操作一直处于阻塞状态，吞吐量通常要高于LinkedBlockingQueue，静态工厂方法Executors.newCachedThreadPool使用了这个队列。
- PriorityBlockingQueue：一个具有优先级的无限阻塞队列。

**maximumPoolSize**（线程池最大大小）：**线程池允许创建的最大线程数**。如果队列满了，并且已创建的线程数小于最大线程数，则线程池会再创建新的线程执行任务。值得注意的是如果使用了无界的任务队列这个参数就没什么效果。

**ThreadFactory**：用于设置创建线程的工厂，可以通过线程工厂给每个创建出来的线程设置更有意义的名字。

**RejectedExecutionHandler**（饱和策略）：**当队列和线程池都满了**，说明线程池处于饱和状态，那么必须采取一种策略处理提交的新任务。这个策略默认情况下是AbortPolicy，表示无法处理新任务时抛出异常。

**keepAliveTime**（线程活动保持时间）：线程池的工作线程空闲后，保持存活的时间。所以如果任务很多，并且每个任务执行的时间比较短，可以调大这个时间，提高线程的利用率。

**TimeUnit**（线程活动保持时间的单位）：可选的单位有天（DAYS），小时（HOURS），分钟（MINUTES），毫秒(MILLISECONDS)，微秒(MICROSECONDS, 千分之一毫秒)和毫微秒(NANOSECONDS, 千分之一微秒)。


#### 线程池的状态: 

RUNNING: 接受新任务，执行队列中的任务。 

SHUTDOWN: 不接受新任务，但是执行队列中的任务。RUNNING->SHUTDOWN，调用shutdown()方法。 

STOP:不接受新任务，不执行队列中的任务，并终止正在执行的任务。RUNNING或者SHUTDOWN->STOP，调用shutdownNow()方法。 

TERMIANTED:同STOP，但是所有线程已经terminated。 

线程池一定要记得shutdown(),这就跟创建了数据库资源finally中记得关闭一样，如Connection ResultSet PreparedStatement Statement，还有文件，网络socket对象，创建后都要记得关闭。 

#### 向线程池提交任务

我们可以使用execute提交的任务，但是execute方法没有返回值，所以无法判断任务是否被线程池执行成功。

		threadsPool.execute(new Runnable() {
	        @Override
	        public void run() {
	            // TODO Auto-generated method stub
	        }
	    });

我们也可以使用submit 方法来提交任务，它会返回一个future,那么我们可以通过这个future来判断任务是否执行成功，通过future的get方法来获取返回值，get方法会阻塞住直到任务完成，而使用get(long timeout, TimeUnit unit)方法则会阻塞一段时间后立即返回，这时有可能任务没有执行完。

		Future<Object> future = executor.submit(harReturnValuetask);
		try {
		     Object s = future.get();
		} catch (InterruptedException e) {
		    // 处理中断异常
		} catch (ExecutionException e) {
		    // 处理无法执行任务异常
		} finally {
		    // 关闭线程池
		    executor.shutdown();
		}







#### 关闭线程池

我们可以通过调用线程池的**shutdown或shutdownNow**方法来关闭线程池。

它们的原理是**遍历**线程池中的工作线程，然后逐个调用线程的interrupt方法来中断线程，所以无法响应中断的任务可能永远无法终止。

但是它们存在一定的区别，shutdownNow首先将线程池的状态设置成STOP，然后尝试停止所有的正在执行或暂停任务的线程，并返回等待执行任务的列表，而shutdown只是将线程池的状态设置成SHUTDOWN状态，然后中断所有没有正在执行任务的线程。

只要调用了这两个关闭方法的其中一个，isShutdown方法就会返回true。

当所有的任务都已关闭后,才表示线程池关闭成功，这时调用isTerminaed方法会返回true。

至于我们应该调用哪一种方法来关闭线程池，应该由提交到线程池的任务特性决定。

通常调用shutdown来关闭线程池，如果任务不一定要执行完，则可以调用shutdownNow。

### 线程池的主要处理流程

![](http://i.imgur.com/qMHjM9S.png)


从上图我们可以看出，当提交一个新任务到线程池时，线程池的处理流程如下：

首先线程池判断**基本线程池（corePoolSize）**是否已满？没满，创建一个工作线程来执行任务。满了，则进入下个流程。
其次线程池判断**工作队列(workQueue)**是否已满？没满，则将新提交的任务存储在工作队列里。满了，则进入下个流程。
最后线程池判断**整个线程池(maximumPoolSize)**是否已满？没满，则创建一个新的工作线程来执行任务，满了，则交给饱和策略来处理这个任务。



#### 拒绝策略

ThreadPoolExecutor中有几个实现RejectedExecutionHandler的内部类： 

AbortPolicy: 这个比较暴力，直接抛异常，拒绝执行。 

DiscardPolicy:这个是个空实现，直接丢弃。 

DiscardOldestPolicy:丢弃队列中的已有的，执行新来的。喜新厌旧啊这个。 

CallerRunsPolicy: 使用调用者线程执行任务。 


### 说说 Executor

下面是Executor 的源码，可以看出来只有一个 execute 方法。

	public interface Executor {
	
	    /**
	     * Executes the given command at some time in the future.  The command
	     * may execute in a new thread, in a pooled thread, or in the calling
	     * thread, at the discretion of the <tt>Executor</tt> implementation.
	     *
	     * @param command the runnable task
	     * @throws RejectedExecutionException if this task cannot be
	     * accepted for execution.
	     * @throws NullPointerException if command is null
	     */
	    void execute(Runnable command);
	}

主要作用：执行已提交的 Runnable 任务的对象。

此接口提供一种将任务提交与每个任务将如何运行的机制（包括线程使用的细节、调度等）分离开来的方法。

通常使用 Executor 而不是显式地创建线程。

例如，可能会使用以下方法，而不是为一组任务中的每个任务调用 new Thread(new(RunnableTask())).start()：

	 Executor executor = anExecutor;
	 executor.execute(new RunnableTask1());
	 executor.execute(new RunnableTask2());
	 ...



#### ExecutorService

	public interface ExecutorService extends Executor {
		...
	}

下面是 ExecutorService的 API。

![](http://i.imgur.com/4AvhIGM.png)

部分重要的方法介绍：

- boolean	**isShutdown**() ： 如果此执行程序已关闭，则返回 true。


- boolean	**isTerminated**() : 如果关闭后所有任务都已完成，则返回 true。

- void	**shutdown**() : 启动一次顺序关闭，执行以前提交的任务，但不接受新任务。


- List<Runnable>	**shutdownNow**() :试图停止所有正在执行的活动任务，暂停处理正在等待的任务，并返回等待执行的任务列表。


- Future<?>	**submit**(Runnable task) : 提交一个 Runnable 任务用于执行，并返回一个表示该任务的 Future。

#### ThreadPoolExecutor 

实现了上面的 ExecutorService 接口。

**创建**：

![](http://i.imgur.com/iYlj4Gb.png)

以上是 ThreadPoolExecutor 的重载构造函数。

参数的含义分别是：

	corePoolSize - 池中所保存的线程数，包括空闲线程。
	maximumPoolSize - 池中允许的最大线程数。
	keepAliveTime - 当线程数大于核心时，此为终止前多余的空闲线程等待新任务的最长时间。
	unit - keepAliveTime 参数的时间单位。
	workQueue - 执行前用于保持任务的队列。此队列仅保持由 execute 方法提交的 Runnable 任务。
	threadFactory - 执行程序创建新线程时使用的工厂。
	handler - 由于超出线程范围和队列容量而使执行被阻塞时所使用的处理程序。

**源码分析**：

execute 方法的源码：

	public void execute(Runnable command) {
	    if (command == null)
	        throw new NullPointerException();
		//如果小于corePoolSize，就通过 addIfUnderCorePoolSize 创建新的线程来处理任务 
	    if (poolSize >= corePoolSize || !addIfUnderCorePoolSize(command)) {
			//如果大于corePoolSize，则优先放入队列中。 
			//workQueue 是一个 BlockingQueue;
	        if (runState == RUNNING && workQueue.offer(command)) {
	            if (runState != RUNNING || poolSize == 0)
	                ensureQueuedTaskHandled(command);
	        }
	        else if (!addIfUnderMaximumPoolSize(command))
	            reject(command); // is shutdown or saturated
	    }
	}
	
	private boolean addIfUnderCorePoolSize(Runnable firstTask) {
	    Thread t = null;
	    final ReentrantLock mainLock = this.mainLock;
	    mainLock.lock();
	    try {
	        if (poolSize < corePoolSize && runState == RUNNING)
				//创建新的线程 
	            t = addThread(firstTask);
	    } finally {
	        mainLock.unlock();
	    }
	    if (t == null)
	        return false;
	    t.start();
	    return true;
	}
	
	private Thread addThread(Runnable firstTask) {
	    Worker w = new Worker(firstTask);
	    Thread t = threadFactory.newThread(w);
	    if (t != null) {
	        w.thread = t;
	        workers.add(w);
	        int nt = ++poolSize;
	        if (nt > largestPoolSize)
	            largestPoolSize = nt;
	    }
	    return t;
	}


注意这里的 addThread 方法，很关键。Worker对象是一个 Runnable 接口，而细看源码会发现，Thread 对象 t 是由 ThreadFactory 的newThread 方法产生的，而该方法的作用只是给 Thread 改了个名字而已。
workers 是一个 HashSet。将包装后的 Thread 加入集合中。

而在 addThread 方法外部，调用了 Thread的 start 方法，那么 Worker 的 run() 方法是什么样的呢？

		public void run() {  
		     try {  
		         Runnable task = firstTask;  
		         firstTask = null;  
		         while (task != null || (task = getTask()) != null) {  
		             runTask(task);  
		             task = null;  
		         }  
		     } finally {  
		         workerDone(this);  
		     }  
		 }  

可以看到其中有一个关键的方法就是 getTask() 方法，可以看一下它的源码：

	Runnable getTask() {
	    for (;;) {
	        try {
	            int state = runState;
	            if (state > SHUTDOWN)
	                return null;
	            Runnable r;
	            if (state == SHUTDOWN)  // Help drain queue
	                r = workQueue.poll();
	            else if (poolSize > corePoolSize || allowCoreThreadTimeOut)
	                r = workQueue.poll(keepAliveTime, TimeUnit.NANOSECONDS);
	            else
	                r = workQueue.take();
	            if (r != null)
	                return r;
	            if (workerCanExit()) {
	                if (runState >= SHUTDOWN) // Wake up others
	                    interruptIdleWorkers();
	                return null;
	            }
	            // Else retry
	        } catch (InterruptedException ie) {
	            // On interruption, re-check runState
	        }
	    }
	}

可以看出来，此方法的作用：

如果线程池的状态不是 RUNNING 或者 SHUTDOWN,则返回。

如果线程池的状态是 SHUTDOWN， 则意味着线程池不接受新的任务，则从 workQueue 这个 BlockingQueue 中取出线程来执行；

如果线程池的状态是 RUNNING，则使用 take 方法取出线程来执行任务。

总而言之，是从阻塞队列中取出线程，后面再通过 runTask() 方法来执行任务。


关于ensureQueuedTaskHandled 源码注释上是这么写的：

> Rechecks state after queuing a task. Called from execute when
> pool state has been observed to change after queuing a task. If
> the task was queued concurrently with a call to shutdownNow,
> and is still present in the queue, this task must be removed
> and rejected to preserve shutdownNow guarantees.  Otherwise,
> this method ensures (unless addThread fails) that there is at
> least one live thread to handle this task

	private void ensureQueuedTaskHandled(Runnable command) {
	    final ReentrantLock mainLock = this.mainLock;
	    mainLock.lock();
	    boolean reject = false;
	    Thread t = null;
	    try {
	        int state = runState;
	        if (state != RUNNING && workQueue.remove(command))
	            reject = true;
	        else if (state < STOP &&
	                 poolSize < Math.max(corePoolSize, 1) &&
	                 !workQueue.isEmpty())
	            t = addThread(null);
	    } finally {
	        mainLock.unlock();
	    }
	    if (reject)
	        reject(command);
	    else if (t != null)
	        t.start();
	}

### 有关使用 ThreadPool的两个例子


例子一：

		package com.java;
		
		import java.util.concurrent.*;
		
		public class JavaStudy {
			public static void main(String[] args) {
				// ExecutorService es = Executors.newCachedThreadPool();
				// ExecutorService es = Executors.newSingleThreadExecutor();
				ExecutorService es = Executors.newFixedThreadPool(2);
		
				for (int i = 0; i < 5; i++) {
					es.execute(new TestRunnable());
					System.out.println("***********a" + i + "**************");
				}
		
				es.shutdown();
			}
		}
		
		class TestRunnable implements Runnable {
			@Override
			public void run() {
				System.out.println(Thread.currentThread().getName() + "线程被调用了");
			}
		}


例子二：




		package com.java;
	
		import java.util.concurrent.*;
		import java.util.*;
	
		public class JavaStudy {
		public static void main(String[] args) {
			ExecutorService es = Executors.newCachedThreadPool();
			// ExecutorService es = Executors.newSingleThreadExecutor();
			//ExecutorService es = Executors.newFixedThreadPool(5);
			List<Future<String>> list = new ArrayList<Future<String>>();
	
			for (int i = 0; i < 20; i++) {
				Future<String> result = es.submit(new TestCallable(i));//注意在使用 Callable 对象作为参数时，只能使用 submit 方法。
				list.add(result);
			}
			
			for(Future<String> res: list) {
				try {
					while(!res.isDone())
						;
					System.out.println(res.get());
				} catch (InterruptedException e) {
					e.printStackTrace();
				} catch (ExecutionException e) {
					e.printStackTrace();
				} finally {
					es.shutdown();
				}
			}
		}
		}
	
		class TestCallable implements Callable<String> {
			
			int id;
			public TestCallable(int id) {
				this.id = id;
			}
			@Override 
			public String call() throws Exception{
				System.out.println(Thread.currentThread().getName() + "线程被调用了");
				return Thread.currentThread().getName() + "线程被调用了, 返回结果 " + id;
			}
		}

运行结果：

		pool-1-thread-1线程被调用了
		pool-1-thread-1线程被调用了, 返回结果 0
		pool-1-thread-3线程被调用了
		pool-1-thread-5线程被调用了
		pool-1-thread-7线程被调用了
		pool-1-thread-9线程被调用了
		pool-1-thread-11线程被调用了
		pool-1-thread-13线程被调用了
		pool-1-thread-15线程被调用了
		pool-1-thread-17线程被调用了
		pool-1-thread-19线程被调用了
		pool-1-thread-2线程被调用了
		pool-1-thread-2线程被调用了, 返回结果 1
		pool-1-thread-3线程被调用了, 返回结果 2
		pool-1-thread-4线程被调用了
		pool-1-thread-4线程被调用了, 返回结果 3
		pool-1-thread-5线程被调用了, 返回结果 4
		pool-1-thread-6线程被调用了
		pool-1-thread-6线程被调用了, 返回结果 5
		pool-1-thread-7线程被调用了, 返回结果 6
		pool-1-thread-8线程被调用了
		pool-1-thread-8线程被调用了, 返回结果 7
		pool-1-thread-9线程被调用了, 返回结果 8
		pool-1-thread-10线程被调用了
		pool-1-thread-10线程被调用了, 返回结果 9
		pool-1-thread-11线程被调用了, 返回结果 10
		pool-1-thread-12线程被调用了
		pool-1-thread-12线程被调用了, 返回结果 11
		pool-1-thread-13线程被调用了, 返回结果 12
		pool-1-thread-14线程被调用了
		pool-1-thread-14线程被调用了, 返回结果 13
		pool-1-thread-16线程被调用了
		pool-1-thread-15线程被调用了, 返回结果 14
		pool-1-thread-16线程被调用了, 返回结果 15
		pool-1-thread-18线程被调用了
		pool-1-thread-17线程被调用了, 返回结果 16
		pool-1-thread-18线程被调用了, 返回结果 17
		pool-1-thread-19线程被调用了, 返回结果 18
		pool-1-thread-20线程被调用了
		pool-1-thread-20线程被调用了, 返回结果 19
例子三

                package com.java;		
    	import java.util.concurrent.*;		
    	public class JavaStudy {
    		public static void main(String[] args) {
    			BlockingQueue<Runnable> bq = new ArrayBlockingQueue<Runnable>(20);
    	
    			ThreadPoolExecutor pool = new ThreadPoolExecutor(3, 5, 50,
    					TimeUnit.MILLISECONDS, bq);
    			for(int i = 0; i < 5; i++) {
    				Runnable t = new TestRunnable();
    				pool.execute(t);
    			}
    			
    			pool.shutdown();


​		
			}
		}
		
		class TestRunnable implements Runnable {
			@Override
			public void run() {
				System.out.println(Thread.currentThread().getName() + "线程被调用了");
			}
		}
