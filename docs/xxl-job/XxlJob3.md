# Executor执行器端执行逻辑

当有调度请求时候会走到Netty的Http服务器经过解码最后执行到自定义ChannelHandler,这个ChannelHandler的实现类是`EmbedHttpServerHandler`，重点‘
来看它的channelRead0方法，源码如下:
```java
 protected void channelRead0(final ChannelHandlerContext ctx, FullHttpRequest msg) throws Exception {

            String requestData = msg.content().toString(CharsetUtil.UTF_8);
            String uri = msg.uri();
            HttpMethod httpMethod = msg.method();
            boolean keepAlive = HttpUtil.isKeepAlive(msg);
            String accessTokenReq = msg.headers().get(XxlJobRemotingUtil.XXL_JOB_ACCESS_TOKEN);

            // invoke
            bizThreadPool.execute(new Runnable() {
                @Override
                public void run() {
                    // do invoke
                    Object responseObj = process(httpMethod, uri, requestData, accessTokenReq);

                    // to json
                    String responseJson = GsonTool.toJson(responseObj);

                    // write response
                    writeResponse(ctx, keepAlive, responseJson);
                }
            });
}
```
主要是调用了process方法，该方法源码如下:
```java
private Object process(HttpMethod httpMethod, String uri, String requestData, String accessTokenReq) {

    if (HttpMethod.POST != httpMethod) {
        return new ReturnT<String>(ReturnT.FAIL_CODE, "invalid request, HttpMethod not support.");
    }
    if (uri==null || uri.trim().length()==0) {
        return new ReturnT<String>(ReturnT.FAIL_CODE, "invalid request, uri-mapping empty.");
    }
    if (accessToken!=null
            && accessToken.trim().length()>0
            && !accessToken.equals(accessTokenReq)) {
        return new ReturnT<String>(ReturnT.FAIL_CODE, "The access token is wrong.");
    }
    try {
        if ("/beat".equals(uri)) {
            return executorBiz.beat();
        } else if ("/idleBeat".equals(uri)) {
            IdleBeatParam idleBeatParam = GsonTool.fromJson(requestData, IdleBeatParam.class);
            return executorBiz.idleBeat(idleBeatParam);
        } else if ("/run".equals(uri)) {
            TriggerParam triggerParam = GsonTool.fromJson(requestData, TriggerParam.class);
            return executorBiz.run(triggerParam);
        } else if ("/kill".equals(uri)) {
            KillParam killParam = GsonTool.fromJson(requestData, KillParam.class);
            return executorBiz.kill(killParam);
        } else if ("/log".equals(uri)) {
            LogParam logParam = GsonTool.fromJson(requestData, LogParam.class);
            return executorBiz.log(logParam);
        } else {
            return new ReturnT<String>(ReturnT.FAIL_CODE, "invalid request, uri-mapping("+ uri +") not found.");
        }
    } catch (Exception e) {
        logger.error(e.getMessage(), e);
        return new ReturnT<String>(ReturnT.FAIL_CODE, "request error:" + ThrowableUtil.toString(e));
    }
}
```
上篇文章说过调度主要是执行的run的url,那就详细看下`executorBiz.run(triggerParam)`执行逻辑，源码如下:
```java
 public ReturnT<String> run(TriggerParam triggerParam) {
        JobThread jobThread = XxlJobExecutor.loadJobThread(triggerParam.getJobId());
        IJobHandler jobHandler = jobThread!=null?jobThread.getHandler():null;
        String removeOldReason = null;
        GlueTypeEnum glueTypeEnum = GlueTypeEnum.match(triggerParam.getGlueType());
        if (GlueTypeEnum.BEAN == glueTypeEnum) {

            // new jobhandler
            IJobHandler newJobHandler = XxlJobExecutor.loadJobHandler(triggerParam.getExecutorHandler());

            // valid old jobThread
            if (jobThread!=null && jobHandler != newJobHandler) {
                // change handler, need kill old thread
                removeOldReason = "change jobhandler or glue type, and terminate the old job thread.";

                jobThread = null;
                jobHandler = null;
            }

            // valid handler
            if (jobHandler == null) {
                jobHandler = newJobHandler;
                if (jobHandler == null) {
                    return new ReturnT<String>(ReturnT.FAIL_CODE, "job handler [" + triggerParam.getExecutorHandler() + "] not found.");
                }
            }

        }
        // executor block strategy
        if (jobThread != null) {
            ExecutorBlockStrategyEnum blockStrategy = ExecutorBlockStrategyEnum.match(triggerParam.getExecutorBlockStrategy(), null);
            if (ExecutorBlockStrategyEnum.DISCARD_LATER == blockStrategy) {
                // discard when running
                if (jobThread.isRunningOrHasQueue()) {
                    return new ReturnT<String>(ReturnT.FAIL_CODE, "block strategy effect："+ExecutorBlockStrategyEnum.DISCARD_LATER.getTitle());
                }
            } else if (ExecutorBlockStrategyEnum.COVER_EARLY == blockStrategy) {
                // kill running jobThread
                if (jobThread.isRunningOrHasQueue()) {
                    removeOldReason = "block strategy effect：" + ExecutorBlockStrategyEnum.COVER_EARLY.getTitle();

                    jobThread = null;
                }
            } else {
                // just queue trigger
            }
        }
        if (jobThread == null) {
            jobThread = XxlJobExecutor.registJobThread(triggerParam.getJobId(), jobHandler, removeOldReason);
        }
        ReturnT<String> pushResult = jobThread.pushTriggerQueue(triggerParam);
        return pushResult;
}
```
这里主要分析常用的Bean运行模式,首先根据job_id从ConcurrentMap<Integer, JobThread> jobThreadRepository中获取JobThread，它是一个线程对象，每一个
job_id对应一个，第一次请求的时候JobThread值是null的，然后根据请求参数`triggerParam.getExecutorHandler()`获取IJobHandler对象，这个逻辑在第一篇文章中
已经分析过了，并把得到的IJobHandler赋值给IJobHandler jobHandler变量，然后调用`XxlJobExecutor.registJobThread(triggerParam.getJobId(), jobHandler, removeOldReason)`方法，该方法源码如下:
```java
    public static JobThread registJobThread(int jobId, IJobHandler handler, String removeOldReason){
        JobThread newJobThread = new JobThread(jobId, handler);
        newJobThread.start();
        logger.info(">>>>>>>>>>> xxl-job regist JobThread success, jobId:{}, handler:{}", new Object[]{jobId, handler});

        JobThread oldJobThread = jobThreadRepository.put(jobId, newJobThread);	// putIfAbsent | oh my god, map's put method return the old value!!!
        if (oldJobThread != null) {
            oldJobThread.toStop(removeOldReason);
            oldJobThread.interrupt();
        }

        return newJobThread;
    }
```
创建JobThread，它的构造方法是这样的:
```java
	public JobThread(int jobId, IJobHandler handler) {
		this.jobId = jobId;
		this.handler = handler;
		this.triggerQueue = new LinkedBlockingQueue<TriggerParam>();
		this.triggerLogIdSet = Collections.synchronizedSet(new HashSet<Long>());
	}
```
封装jobId、IJobHandler，实例化一个成员属性LinkedBlockingQueue的队列，实例化一个成员属性Set<Long>，接着启动线程，把jobId和对应的线程放到
`ConcurrentMap<Integer, JobThread>`上。

接着调用` jobThread.pushTriggerQueue(triggerParam)`方法，该方法源码如下:
```java
	public ReturnT<String> pushTriggerQueue(TriggerParam triggerParam) {
		// avoid repeat
		if (triggerLogIdSet.contains(triggerParam.getLogId())) {
			logger.info(">>>>>>>>>>> repeate trigger job, logId:{}", triggerParam.getLogId());
			return new ReturnT<String>(ReturnT.FAIL_CODE, "repeate trigger job, logId:" + triggerParam.getLogId());
		}

		triggerLogIdSet.add(triggerParam.getLogId());
		triggerQueue.add(triggerParam);
        return ReturnT.SUCCESS;
	}
```
triggerLogIdSet的Set集合判断Log_id是否已经存在如果存在说明是重复调度返回错误异常，如果不是添加到triggerLogIdSet中，把TriggerParam添加到
等待队列这种，返回调度成功的状态，现在主要来看看JobThread的run方法，它是执行调度的关键。

```java
public void run() {

    	// init
    	try {
			handler.init();
		} catch (Throwable e) {
    		logger.error(e.getMessage(), e);
		}

		// execute
		while(!toStop){
			running = false;
			idleTimes++;

            TriggerParam triggerParam = null;
            try {
				// to check toStop signal, we need cycle, so wo cannot use queue.take(), instand of poll(timeout)
				triggerParam = triggerQueue.poll(3L, TimeUnit.SECONDS);
				if (triggerParam!=null) {
					running = true;
					idleTimes = 0;
					triggerLogIdSet.remove(triggerParam.getLogId());

					// log filename, like "logPath/yyyy-MM-dd/9999.log"
					String logFileName = XxlJobFileAppender.makeLogFileName(new Date(triggerParam.getLogDateTime()), triggerParam.getLogId());
					XxlJobContext xxlJobContext = new XxlJobContext(
							triggerParam.getJobId(),
							triggerParam.getExecutorParams(),
							logFileName,
							triggerParam.getBroadcastIndex(),
							triggerParam.getBroadcastTotal());

					// init job context
					XxlJobContext.setXxlJobContext(xxlJobContext);

					// execute
					XxlJobHelper.log("<br>----------- xxl-job job execute start -----------<br>----------- Param:" + xxlJobContext.getJobParam());

					if (triggerParam.getExecutorTimeout() > 0) {
						// limit timeout
						Thread futureThread = null;
						try {
							FutureTask<Boolean> futureTask = new FutureTask<Boolean>(new Callable<Boolean>() {
								@Override
								public Boolean call() throws Exception {

									// init job context
									XxlJobContext.setXxlJobContext(xxlJobContext);

									handler.execute();
									return true;
								}
							});
							futureThread = new Thread(futureTask);
							futureThread.start();

							Boolean tempResult = futureTask.get(triggerParam.getExecutorTimeout(), TimeUnit.SECONDS);
						} catch (TimeoutException e) {

							XxlJobHelper.log("<br>----------- xxl-job job execute timeout");
							XxlJobHelper.log(e);

							// handle result
							XxlJobHelper.handleTimeout("job execute timeout ");
						} finally {
							futureThread.interrupt();
						}
					} else {
						// just execute
						handler.execute();
					}

					// valid execute handle data
					if (XxlJobContext.getXxlJobContext().getHandleCode() <= 0) {
						XxlJobHelper.handleFail("job handle result lost.");
					} else {
						String tempHandleMsg = XxlJobContext.getXxlJobContext().getHandleMsg();
						tempHandleMsg = (tempHandleMsg!=null&&tempHandleMsg.length()>50000)
								?tempHandleMsg.substring(0, 50000).concat("...")
								:tempHandleMsg;
						XxlJobContext.getXxlJobContext().setHandleMsg(tempHandleMsg);
					}
					XxlJobHelper.log("<br>----------- xxl-job job execute end(finish) -----------<br>----------- Result: handleCode="
							+ XxlJobContext.getXxlJobContext().getHandleCode()
							+ ", handleMsg = "
							+ XxlJobContext.getXxlJobContext().getHandleMsg()
					);

				} else {
					if (idleTimes > 30) {
						if(triggerQueue.size() == 0) {	// avoid concurrent trigger causes jobId-lost
							XxlJobExecutor.removeJobThread(jobId, "excutor idel times over limit.");
						}
					}
				}
			} catch (Throwable e) {
				if (toStop) {
					XxlJobHelper.log("<br>----------- JobThread toStop, stopReason:" + stopReason);
				}

				// handle result
				StringWriter stringWriter = new StringWriter();
				e.printStackTrace(new PrintWriter(stringWriter));
				String errorMsg = stringWriter.toString();

				XxlJobHelper.handleFail(errorMsg);

				XxlJobHelper.log("<br>----------- JobThread Exception:" + errorMsg + "<br>----------- xxl-job job execute end(error) -----------");
			} finally {
                if(triggerParam != null) {
                    // callback handler info
                    if (!toStop) {
                        // commonm
                        TriggerCallbackThread.pushCallBack(new HandleCallbackParam(
                        		triggerParam.getLogId(),
								triggerParam.getLogDateTime(),
								XxlJobContext.getXxlJobContext().getHandleCode(),
								XxlJobContext.getXxlJobContext().getHandleMsg() )
						);
                    } else {
                        // is killed
                        TriggerCallbackThread.pushCallBack(new HandleCallbackParam(
                        		triggerParam.getLogId(),
								triggerParam.getLogDateTime(),
								XxlJobContext.HANDLE_COCE_FAIL,
								stopReason + " [job running, killed]" )
						);
                    }
                }
            }
        }

		// callback trigger request in queue
		while(triggerQueue !=null && triggerQueue.size()>0){
			TriggerParam triggerParam = triggerQueue.poll();
			if (triggerParam!=null) {
				// is killed
				TriggerCallbackThread.pushCallBack(new HandleCallbackParam(
						triggerParam.getLogId(),
						triggerParam.getLogDateTime(),
						XxlJobContext.HANDLE_COCE_FAIL,
						stopReason + " [job not executed, in the job queue, killed.]")
				);
			}
		}

		// destroy
		try {
			handler.destroy();
		} catch (Throwable e) {
			logger.error(e.getMessage(), e);
		}

		logger.info(">>>>>>>>>>> xxl-job JobThread stoped, hashCode:{}", Thread.currentThread());
	}
```
如果XxlJob注解指定init方法则反射调用目标对象init方法,从等待队列中取出调度参数对象TriggerParam，判断如果不为空，创建一个XxlJobContext对象用于保存调度时候指定的参数然后把这个对象放入到
ThreadLocal中这样业务方法可以取得页面指定的参数值。

判断调度页面是否指定了任务超时时间，如果指定了创建一个FutureTask任务执行业务操作并且看在指定时间是否可以返回结果，如果返回不了记录错误日志并且中断
FutureTask对应的任务线程。

如果没指定任务超时时间就正常反射调用目标对象方法。

如果从队列中取不到TriggerParam对象，执行这段分支逻辑:
```java
if (idleTimes > 30) {
    if(triggerQueue.size() == 0) {	// avoid concurrent trigger causes jobId-lost
        XxlJobExecutor.removeJobThread(jobId, "excutor idel times over limit.");
    }
}
```
idleTimes表示 JobThread 连续空闲的次数如果大于30次并且确认当前队列是空的，自动回收长期空闲的JobThread，防止执行器因为历史job越来越多而线程无限增长。

catch块里面记录业务方法执行是否发生异常如果发生异常记录异常日志，把执行handleCode(响应码)、handleMsg(异常信息)设置到XxlJobContext中并记录本地错误日志。

finally代码块执行的是无论成功还是失败都要回调给Admin进行日志处理，它的代码逻辑如下:
```java
 TriggerCallbackThread.pushCallBack(new HandleCallbackParam(
        triggerParam.getLogId(),
        triggerParam.getLogDateTime(),
        XxlJobContext.getXxlJobContext().getHandleCode(),
        XxlJobContext.getXxlJobContext().getHandleMsg() )
        );
```
把HandleCallbackParam对象添加对象TriggerCallbackThread的成员变量LinkedBlockingQueue<HandleCallbackParam>上，然后triggerCallbackThread线程异步从队列取
出来回调给Admin端，Admin端处理逻辑详细看上篇文章。

如果线程被停止同样执行上面逻辑，只不过handleMsg的值是`job running, killed`。

如果线程停止瞬间判断triggerQueue是否还有元素如果有就继续执行pushCallBack方法，只不过handleMsg的值是`job not executed, in the job queue, killed`。

最后执行`handler.destroy()`方法调用我们业务对象指定的destroy对应的方法。

在`ExecutorBizImpl.run`方法还有一段分支逻辑就是多次调用JobThread不为null的情况，它的源码如下:
```java
if (jobThread != null) {
    ExecutorBlockStrategyEnum blockStrategy = ExecutorBlockStrategyEnum.match(triggerParam.getExecutorBlockStrategy(), null);
    if (ExecutorBlockStrategyEnum.DISCARD_LATER == blockStrategy) {
        // discard when running
        if (jobThread.isRunningOrHasQueue()) {
            return new ReturnT<String>(ReturnT.FAIL_CODE, "block strategy effect："+ExecutorBlockStrategyEnum.DISCARD_LATER.getTitle());
        }
    } else if (ExecutorBlockStrategyEnum.COVER_EARLY == blockStrategy) {
        // kill running jobThread
        if (jobThread.isRunningOrHasQueue()) {
            removeOldReason = "block strategy effect：" + ExecutorBlockStrategyEnum.COVER_EARLY.getTitle();

            jobThread = null;
        }
    } else {
        // just queue trigger
    }
}
```
这里的处理逻辑涉及到调度页面的阻塞处理策略的设置，它们的策略分别都有如下图所示:

![xxl-job1.png](..%2Fimg%2Fxxl-job1.png)

主要是分为三种单机串行、丢弃后续调度、覆盖之前调度，下面分别来看它们实现逻辑：

按照源码顺序来看先判断是否是丢弃后续调度，如果同一个 job 当前正在运行，新的触发请求直接丢弃，不排队,这个运行判断标准是根据running是true或者
队列不为空来判断的，如果条件是true返回一个错误提示给Admin。

如果是覆盖之前调度，同样是判断同一个job当前是否正在运行，如果是赋值`removeOldReason = "block strategy effect：" + ExecutorBlockStrategyEnum.COVER_EARLY.getTitle();`表示
老的调度线程被销毁的原因，把jobThread引用赋值为null，然后程序继续执行的时候执行这段代码`jobThread = XxlJobExecutor.registJobThread(triggerParam.getJobId(), jobHandler, removeOldReason)`就是前面首次
创建线程的逻辑同时把老的线程任务打断回收掉。

上面条件都不成立的话就是单机串行，这个逻辑就比较简单了就是加入到队列中等待下次运行。