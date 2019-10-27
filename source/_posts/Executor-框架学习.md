---
title: Executor 框架学习
tags:
  - Concurrency
date: 2015-09-02 10:59:08
categories: 技术
---

![](http://i.imgur.com/arAfeHW.png)

上图是 Executor 框架的轮廓图。

下面进行一一介绍：

### Future

	public interface Future<V> {
		boolean	cancel(boolean mayInterruptIfRunning) ;//如果应该中断执行此任务的线程，则为 true；否则允许正在运行的任务运行完成
 		V get(); //如有必要，等待计算完成，然后获取其结果。
		V get(long timeout, TimeUnit unit);//如有必要，最多等待所给定的时间之后，获取其结果（如果结果可用）。
		boolean	isCancelled();//如果在任务正常完成前将其取消，则返回 true。
		boolean	isDone();//如果任务已完成，则返回 true。
	}

实现类：

	FutureTask, SwingWorker

Future 表示异步计算的结果。它提供了检查计算是否完成的方法，以等待计算的完成，并获取计算的结果。

计算完成后只能使用 get 方法来获取结果，如有必要，计算完成前可以阻塞此方法。

取消则由 cancel 方法来执行。还提供了其他方法，以确定任务是正常完成还是被取消了。一旦计算完成，就不能再取消计算。如果为了可取消性而使用 Future 但又不提供可用的结果，则可以声明 Future<?> 形式类型、并返回 null 作为底层任务的结果。

#### FutureTask

	public class FutureTask<V>  implements RunnableFuture<V> {

	}

	//可使用 FutureTask 包装 Callable 或 Runnable 对象。因为 FutureTask 实现了 Runnable，所以可将 FutureTask 提交给 Executor 执行。
     
     
重要的构造方法：

	FutureTask(Callable<V> callable) 
	//创建一个 FutureTask，一旦运行就执行给定的 Callable。
	FutureTask(Runnable runnable, V result) 
	//创建一个 FutureTask，一旦运行就执行给定的 Runnable，并安排成功完成时 get 返回给定的结果 。
     
### callable

	public interface Callable<V> {
		V call();
		 // 只有这一个方法
         // 计算结果，如果无法计算结果，则抛出一个异常。
	}
      

### Executor

	public interface Executor {
		void execute(Runnable command); 
          //在未来某个时间执行给定的命令。
	}


Executor 使用 Runnable 作为其任务的基本表达形式。

此接口提供一种将任务提交与每个任务将如何运行的机制（包括线程使用的细节、调度等）分离开来的方法。通常使用 Executor 而不是显式地创建线程。

例如，可能会使用以下方法

	 Executor executor = anExecutor;
	 executor.execute(new RunnableTask1());
	 executor.execute(new RunnableTask2());
	 ...
而不是为一组任务中的每个任务调用 

	 new Thread(new(RunnableTask())).start()：

不过， Executor 接口并没有严格地要求执行是异步的。在最简单的情况下，执行程序可以在调用者的线程中立即运行已提交的任务：

	 class DirectExecutor implements Executor {
	     public void execute(Runnable r) {
	         r.run();
	     }
	 }

更常见的是，任务是在某个不是调用者线程的线程中执行的。以下执行程序将为每个任务生成一个新线程。
	 class ThreadPerTaskExecutor implements Executor {
	     public void execute(Runnable r) {
	         new Thread(r).start();
	     }
	 }

子接口： ExecutorService

		public interface ExecutorService extends Executor {

		}
下面是其API轮廓图：

![](http://i.imgur.com/w0AaBM1.png)

可以看出来，其继承 Executor 后在其基础上又扩展了很多方法。

实现子类：ThreadPoolExecutor

构造方法：

	public ThreadPoolExecutor(int corePoolSize,
	                          int maximumPoolSize,
	                          long keepAliveTime,
	                          TimeUnit unit,
	                          BlockingQueue<Runnable> workQueue,
	                          ThreadFactory threadFactory,
	                          RejectedExecutionHandler handler)

用给定的初始参数创建新的 ThreadPoolExecutor。

参数：

	corePoolSize - 池中所保存的线程数，包括空闲线程。
	maximumPoolSize - 池中允许的最大线程数。
	keepAliveTime - 当线程数大于核心时，此为终止前多余的空闲线程等待新任务的最长时间。
	unit - keepAliveTime 参数的时间单位。
	workQueue - 执行前用于保持任务的队列。此队列仅保持由 execute 方法提交的 Runnable 任务。
	threadFactory - 执行程序创建新线程时使用的工厂。
	handler - 由于超出线程范围和队列容量而使执行被阻塞时所使用的处理程序。

抛出：

	IllegalArgumentException - 如果 corePoolSize 或 keepAliveTime 小于 0，或者 maximumPoolSize 小于等于 0，或者 corePoolSize 大于 maximumPoolSize。
	NullPointerException - 如果 workQueue、 threadFactory 或 handler 为 null。

### CompletionService

	public interface CompletionService<V> {

	}

**将生产新的异步任务与使用已完成任务的结果分离开来的服务。**

在此之前，如果你向 Executor 提交了一个批处理任务，并且希望在它们完成后获得结果，为此你可以保存与每个任务相关联的 Future，然后不断地调用 timeout 为 0 的get，来检验 Future 是否完成。这样做略显麻烦。

这个问题，用CompletionService来完成。

生产者 submit 执行的任务。使用者 take 已完成的任务，并按照完成这些任务的顺序处理它们的结果。

例如，CompletionService 可以用来管理异步 IO ，执行读操作的任务作为程序或系统的一部分提交，然后，当完成读操作时，会在程序的不同部分执行其他操作，执行操作的顺序可能与所请求的顺序不同。

通常，CompletionService 依赖于一个单独的 Executor 来实际执行任务，在这种情况下，CompletionService **只管理一个内部完成队列**。ExecutorCompletionService 类提供了此方法的一个实现。

方法摘要：

	 Future<V>	poll() 
	          获取并移除表示下一个已完成任务的 Future，如果不存在这样的任务，则返回 null。
	 Future<V>	poll(long timeout, TimeUnit unit) 
	          获取并移除表示下一个已完成任务的 Future，如果目前不存在这样的任务，则将等待指定的时间（如果有必要）。
	 Future<V>	submit(Callable<V> task) 
	          提交要执行的值返回任务，并返回表示挂起的任务结果的 Future。
	 Future<V>	submit(Runnable task, V result) 
	          提交要执行的 Runnable 任务，并返回一个表示任务完成的 Future，可以提取或轮询此任务。
	 Future<V>	take() 
	          获取并移除表示下一个已完成任务的 Future，如果目前不存在这样的任务，则等待。

实现子类：

	public class ExecutorCompletionService<V> extends Object implements CompletionService<V> {
	
	}

它的实现原理是这样的：

它在构造函数中创建了一个 BlockingQueue，用它来保存完成的结果。计算完成时会调用FutureTask中的done方法。

当提交了一个任务后，首先把这个任务包装为一个 QueuingFuture，它是FutureTask的一个子类，覆盖了done方法。将结果置入 BlockingQueue中。

take 和 poll 方法委托给了 BlockingQueue。


用法示例：

 假定您有针对某个问题的一组求解程序，每个求解程序都能返回某种类型的 Result 值，并且您想同时运行它们，使用方法 use(Result r) 处理返回非 null 值的每个求解程序的返回结果。可以这样编写程序：

    void solve(Executor e, Collection<Callable<Result>> solvers)
      throws InterruptedException, ExecutionException {
        CompletionService<Result> ecs = new ExecutorCompletionService<Result>(e);
        for (Callable<Result> s : solvers)
            ecs.submit(s);
        int n = solvers.size();
        for (int i = 0; i < n; ++i) {
            Result r = ecs.take().get();
            if (r != null) 
                use(r);
        }
    }
