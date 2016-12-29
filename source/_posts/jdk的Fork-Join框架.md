---
title: jdk的Fork/Join框架
date: 2016-12-27 15:59:26
tags: 
 - concurrent
categories:
 - concurrent
---

jkd7提供了一个用于并行执行任务的框架，是把一个大任务分割成若干个小任务，将小任务的执行结果汇总起来，得到大任务的结果。
<!-- more -->
```java
public class ForkJoinTaskDemo {
	public static void main(String[] args) throws Exception {
		ForkJoinPool forkJoinPool = new ForkJoinPool();

		CountTask task = new CountTask(1, 5);
		Future<Integer> result = forkJoinPool.submit(task);
		System.out.println("1-5相加结果：" + result.get());

		CountTask task2 = new CountTask(1, 100);
		Future<Integer> result2 = forkJoinPool.submit(task2);
		System.out.println("1-100相加结果：" + result2.get());

		System.out.println("thread main end!");

	}
}

public class CountTask extends RecursiveTask<Integer> {
	private static int selitSize = 2;
	private int start, end;

	public CountTask(int start, int end) {
		this.start = start;
		this.end = end;
	}

	@Override
	protected Integer compute() {
		int sum = 0;
		// 如果任务不需要再拆分 就开始计算
		boolean canCompute = (end - start) <= selitSize;
		if (canCompute) {
			for (int i = start; i <= end; i++) {
				sum += i;
			}
		} else {
			// 拆分成两个子任务
			int middle = (start + end) / 2;
			CountTask firstTask = new CountTask(start, middle);
			CountTask secondTask = new CountTask(middle + 1, end);

			// 开始执行任务
			firstTask.fork();
			secondTask.fork();

			// 获取第一个任务的执行结果，得不到结果，此线程不会往下面执行。
			Integer firstResult = firstTask.join();
			Integer secoundResult = secondTask.join();

			// 合并两个儿子的执行结果
			sum = firstResult + secoundResult;
		}
		return sum;
	}
}

```

执行结果：

	1-5相加结果：15
	1-100相加结果：5050
	thread main end!

ForkJoinTask与一般的任务的主要区别在于它需要实现`compute`方法，在这个方法，首先需要判断任务是否足够小，如果足够小就直接执行任务。如果任务不足够小，就必须分割成两个子任务，每个子任务在调用`fork`方法时，又会进入`compute`方法，当前子任务如果不需要再继续分割，则执行当前子任务并返回结果。使用join方法会等待子任务执行完成并得到其结果。

`fork()`：这个方法决定了ForkJoinTask的异步执行，凭借这个方法可以创建新的任务。
`join()`：该方法负责在计算完成后返回结果，因此允许一个任务等待另一个任务执行完成。

Fork/Join的执行逻辑是这样：
第一步分割任务。需要有一个fork来把大任务分割成子任务，分割后的任务可能很大，还需要继续分割，直到分割的子任务足够小；分割的任务都放在当前线程所维护的双端队列中，进入队列的头部。
第二步执行任务并得到结果。执行任务时，几个启动线程从队列中获取任务并执行，子任务执行完的结果统一放在一个队列里，启动一个线程从队列里拿数据，合并这些数据。


通常情况下我们不需要直接继承ForkJionTask，只需要继承它的子类，重载`protected void coumpute()`方法。ForkJionTask提供了两个子类：
`RecursiveAction`：用于没有返回结果的任务。
`RecursiveTask`：用于有返回结果的任务。

使用Fork/Jion统计某盘符下的文件数量：
```java
public class ForkJoinTaskDemo {
	private static final String DIR = "D:/creditease";

	public static void main(String[] args) throws Exception {
		CountingTask task = new CountingTask(Paths.get(DIR));
		Integer count = new ForkJoinPool().invoke(task);
		System.out.println(DIR + "下面总文件数量：" + count);
	}
}

public class CountingTask extends RecursiveTask<Integer>{
	private Path dir;

	public CountingTask(Path dir) {
		this.dir = dir;
	}

	@Override
	protected Integer compute() {
		int count = 0; // 文件数
		List<CountingTask> subTasks = new ArrayList<>();
		try {
			// 读取目录dir的子路径。
			DirectoryStream<Path> ds = Files.newDirectoryStream(dir);
			for (Path subPath : ds) {
				if (Files.isDirectory(subPath, LinkOption.NOFOLLOW_LINKS)) {
					subTasks.add(new CountingTask(subPath));
				} else {
					count++;
				}
			}

			if (!subTasks.isEmpty()) {
				// 在当前的ForkJoinPool上高度所有的子任务
				for (CountingTask subTask : invokeAll(subTasks)) {
					count += subTask.join();
				}
			}
		} catch (Exception e) {
			e.printStackTrace();
			return 0;
		}

		return count;
	}

}
```
对于树形结构的遍历处理非常适合。

简单总结下：
> 使用Fork/Jion模式，可以方便的实现并发任务的拆分，不需要处理各种相关事务，同步、通信之类的，仅仅关注如何划分和组合中间结果。