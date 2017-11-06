# AsynTask #

[关于Runnable，Callable与FutureTask](http://blog.csdn.net/yangzhaomuma/article/details/51722779)

本文主要介绍了AsynTask的大致流程

## 状态 ##

	// 默认值
	private volatile Status mStatus = Status.PENDING;

	public enum Status {
        /**
         * Indicates that the task has not been executed yet.
         */
        PENDING,
        /**
         * Indicates that the task is running.
         */
        RUNNING,
        /**
         * Indicates that {@link AsyncTask#onPostExecute} has finished.
         */
        FINISHED,
    }


## 线程池 ##


### THREAD_POOL_EXECUTOR ###

里面是静态的有个线程池THREAD_POOL_EXECUTOR,可以理解为工作线程池

    public static final Executor THREAD_POOL_EXECUTOR;


	static {
        ThreadPoolExecutor threadPoolExecutor = new ThreadPoolExecutor(
                CORE_POOL_SIZE, MAXIMUM_POOL_SIZE, KEEP_ALIVE_SECONDS, TimeUnit.SECONDS,
                sPoolWorkQueue, sThreadFactory);
        threadPoolExecutor.allowCoreThreadTimeOut(true);
        THREAD_POOL_EXECUTOR = threadPoolExecutor;
    }

- CORE_POOL_SIZE ： 核心数目 最少2个，最多4个，根据CPU决定

		// 获取到当前的cpu数目
	    private static final int CPU_COUNT = Runtime.getRuntime().availableProcessors();

		// 最少2个，最多4个，可能会有3个
		private static final int CORE_POOL_SIZE = Math.max(2, Math.min(CPU_COUNT - 1, 4));


- MAXIMUM_POOL_SIZE：最大线程数目为 CPU_COUNT * 2 + 1个
		
	    private static final int MAXIMUM_POOL_SIZE = CPU_COUNT * 2 + 1;

- KEEP_ALIVE_SECONDS：存活时间 TimeUnit.SECONDS

	    private static final int KEEP_ALIVE_SECONDS = 30;

- sPoolWorkQueue：任务队列是个容量大小为128的LinkedBlockingQueue

	private static final BlockingQueue<Runnable> sPoolWorkQueue =
            new LinkedBlockingQueue<Runnable>(128);

- sThreadFactory：创建线程

		private static final ThreadFactory sThreadFactory = new ThreadFactory() {
     	   private final AtomicInteger mCount = new AtomicInteger(1);

     	   public Thread newThread(Runnable r) {
      	      return new Thread(r, "AsyncTask #" + mCount.getAndIncrement());
    	    }
   		};


### sDefaultExecutor ###

这个线程池主要是给我们维护了一个队列进行不断执行取出task进行执行

[ArrayDeque介绍](http://www.cnblogs.com/mthoutai/p/7371602.html)

	 private static class SerialExecutor implements Executor {
		// ArrayDeque 是一种双端队列
        final ArrayDeque<Runnable> mTasks = new ArrayDeque<Runnable>();
		// 当前任务
        Runnable mActive;

        public synchronized void execute(final Runnable r) {
			// 把这个任务加到队列的尾端
            mTasks.offer(new Runnable() {
                public void run() {
                    try {
						// 执行mWork的Call方法
                        r.run();
                    } finally { // 执行完后继续执行下一个
                        scheduleNext();
                    }
                }
            });
			// 如果当前是没任务的话，就执行下一个
            if (mActive == null) {
                scheduleNext();
            }
        }

        protected synchronized void scheduleNext() {
			//不断取出来，不断执行
            if ((mActive = mTasks.poll()) != null) {
				// 最终的执行还是用THREAD_POOL_EXECUTOR线程的方法
                THREAD_POOL_EXECUTOR.execute(mActive);
            }
        }
    }

## 构造方法看 ##

	public AsyncTask(@Nullable Looper callbackLooper) {
		// 默认的loop是主线程，handler是自己new出来的InternalHandler
        mHandler = callbackLooper == null || callbackLooper == Looper.getMainLooper()
            ? getMainHandler()
            : new Handler(callbackLooper);

		// 这里定义了mWorker,mWorker就是实现了Callable的抽象类
        mWorker = new WorkerRunnable<Params, Result>() {
            public Result call() throws Exception {
                mTaskInvoked.set(true);
                Result result = null;
                try {
					// 设置线程优先级别
                    Process.setThreadPriority(Process.THREAD_PRIORITY_BACKGROUND);
                    // doInBackground 我们熟悉的方法，注意是线程中
					// publishProgress需要我们doInBackground在该方法中
					//自己加入我们想要的value，不然无法调用
                    result = doInBackground(mParams);
                   ...
				finally { /// 最终调用postResult方法
          	          postResult(result);
          	      }
                return result;
            }
        };

        mFuture = new FutureTask<Result>(mWorker) {
            @Override
            protected void done() {
                try {
					// 如果mFuture关闭后，执行完后的该方法
                    postResultIfNotInvoked(get());
               ....
            }
        };
    }

postResultIfNotInvoked

	private void postResultIfNotInvoked(Result result) {
        final boolean wasTaskInvoked = mTaskInvoked.get();
		// 如果此时该任务不是工作中
        if (!wasTaskInvoked) { // 一般是true 不会执行，可能某些特殊原因
			// 执行postResult方法
            postResult(result);
        }
    }

postResult：

	// 这个方法很明显最终是发送消息，给handler，进行完成的方法
	// 这里的obj是个AsyncTaskResult<Result>(this, result)
	private Result postResult(Result result) {
        @SuppressWarnings("unchecked")
        Message message = getHandler().obtainMessage(MESSAGE_POST_RESULT,
                new AsyncTaskResult<Result>(this, result));
        message.sendToTarget();
        return result;
    }



## execute ##

接着看调用的execute方法

	@MainThread
    public final AsyncTask<Params, Progress, Result> execute(Params... params) {
		// 这里传入了参数和默认线程池，最终调用了executeOnExecutor
        return executeOnExecutor(sDefaultExecutor, params);
    }

进入executeOnExecutor方法   


	@MainThread
    public final AsyncTask<Params, Progress, Result> executeOnExecutor(Executor exec,
            Params... params) {
        if (mStatus != Status.PENDING) { //如果状态不是PENDING，报错
            switch (mStatus) {
                case RUNNING:
                    throw new IllegalStateException("Cannot execute task:"
                            + " the task is already running.");
                case FINISHED:
                    throw new IllegalStateException("Cannot execute task:"
                            + " the task has already been executed "
                            + "(a task can be executed only once)");
            }
        }
		// 更改状态
        mStatus = Status.RUNNING;
		// 回调给外面，此时该方法还在主线程
        onPreExecute();
		// mWoker是什么？ 是实现Callable的一个抽象类
        mWorker.mParams = params;

		// 最后调用execute方法
		// 如果是默认的此时可以查看上文的sDefaultExecutor的执行
        exec.execute(mFuture);

        return this;
    }

这里提一下publishProgress方法，进度回调，需要我们在doInBackground方法自己加入，否则不会调用

	@WorkerThread
    protected final void publishProgress(Progress... values) {
        if (!isCancelled()) {
            getHandler().obtainMessage(MESSAGE_POST_PROGRESS,
                    new AsyncTaskResult<Progress>(this, values)).sendToTarget();
        }
    }

### InternalHandler ###

最终，相关UI更新，进度更新，任务回调都是放在这个handler，前提是你构造方法的时候没有使用别的handler

		private static class InternalHandler extends Handler {
        public InternalHandler(Looper looper) {
            super(looper);
        }

        @SuppressWarnings({"unchecked", "RawUseOfParameterizedType"})
        @Override
        public void handleMessage(Message msg) {
            AsyncTaskResult<?> result = (AsyncTaskResult<?>) msg.obj;
            switch (msg.what) {
                case MESSAGE_POST_RESULT:
                    // 结果返回
					// 调用 该asynntask的finish方法
                    result.mTask.finish(result.mData[0]);
                    break;
                case MESSAGE_POST_PROGRESS:
					//进度条更新
                    result.mTask.onProgressUpdate(result.mData);
                    break;
            }
        }
    }


finsih方法：代码很清晰,  
从最后面关闭我们也看出如果状态至为FINISHED，而不是PENDING  
说明AsynTask设计上只希望我们execute一次
		
		private void finish(Result result) {
        if (isCancelled()) {
            onCancelled(result);
        } else {
            onPostExecute(result);
        }
        mStatus = Status.FINISHED; // 更改状态
    }

## 总结 ##

看完大致流程，小小的总结一下：

AsynTask其实真正执行操作的线程池是THREAD_POOL_EXECUTOR，而sDefaultExecutor只是维持任务队列的，但是我们可以使用executeOnExecutor方法来构建我们自己的线程池，最终利用Handler更新UI。

AsynTask的优点与缺点：

简单,快捷
过程可控      
使用的缺点:
在使用多个异步操作和并需要进行Ui变更时,就变得复杂起来.







