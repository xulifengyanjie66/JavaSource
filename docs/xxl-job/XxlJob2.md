# XXL-JOB Admin端启动流程和调度算法源码

## 一、前言

XXL-JOB 作为国内使用最广泛的轻量级分布式任务调度平台，其 Admin 端（调度中心） 的设计极具代表性。

本文将深入到 XXL-JOB Admin 端的源码层面，完整梳理其启动流程与核心调度链路。从 XxlJobScheduler 的初始化，到 registryMonitorThread 的注册保活机制；从 慢任务探测与线程池隔离 的时间滑动窗口算法，到 十种路由策略（轮询、LRU、LFU、一致性 Hash 等）的底层实现，逐一拆解。

## 二、Admin端启动源码分析

Admin端SpringBoot启动时候有个@Component对象的XxlJobAdminConfig，它实现了InitializingBean接口，会自动调用它的afterPropertiesSet方法，该方法主要是实例化一个XxlJobScheduler对象，并调用它的init方法，该方法源码如下:

```java
public void init() throws Exception {
        // admin trigger pool start
        JobTriggerPoolHelper.toStart();

        // admin registry monitor run
        JobRegistryHelper.getInstance().start();
        
}
```
首先调用`JobTriggerPoolHelper.toStart()`方法初始化二个线程池对象分别是ThreadPoolExecutor fastTriggerPool、ThreadPoolExecutor slowTriggerPool对象，其中fastTriggerPool用于执行正常调度，slowTriggerPool用于执行
慢调度就是执行时间比较长的，具体算法我会在调度源码分析。


然后调用`JobRegistryHelper.getInstance().start()`方法初始化一个ThreadPoolExecutor registryOrRemoveThreadPool线程池对象，它主要适用于
处理Executor端注册或者移除注册的线程池对象，然后启动一个常驻线程，它的源码如下:
```java
registryMonitorThread = new Thread(new Runnable() {
			@Override
			public void run() {
				while (!toStop) {
					try {
						// auto registry group
						List<XxlJobGroup> groupList = XxlJobAdminConfig.getAdminConfig().getXxlJobGroupDao().findByAddressType(0);
						if (groupList!=null && !groupList.isEmpty()) {

							// remove dead address (admin/executor)
							List<Integer> ids = XxlJobAdminConfig.getAdminConfig().getXxlJobRegistryDao().findDead(RegistryConfig.DEAD_TIMEOUT, new Date());
							if (ids!=null && ids.size()>0) {
								XxlJobAdminConfig.getAdminConfig().getXxlJobRegistryDao().removeDead(ids);
							}

							// fresh online address (admin/executor)
							HashMap<String, List<String>> appAddressMap = new HashMap<String, List<String>>();
							List<XxlJobRegistry> list = XxlJobAdminConfig.getAdminConfig().getXxlJobRegistryDao().findAll(RegistryConfig.DEAD_TIMEOUT, new Date());
							if (list != null) {
								for (XxlJobRegistry item: list) {
									if (RegistryConfig.RegistType.EXECUTOR.name().equals(item.getRegistryGroup())) {
										String appname = item.getRegistryKey();
										List<String> registryList = appAddressMap.get(appname);
										if (registryList == null) {
											registryList = new ArrayList<String>();
										}

										if (!registryList.contains(item.getRegistryValue())) {
											registryList.add(item.getRegistryValue());
										}
										appAddressMap.put(appname, registryList);
									}
								}
							}

							// fresh group address
							for (XxlJobGroup group: groupList) {
								List<String> registryList = appAddressMap.get(group.getAppname());
								String addressListStr = null;
								if (registryList!=null && !registryList.isEmpty()) {
									Collections.sort(registryList);
									StringBuilder addressListSB = new StringBuilder();
									for (String item:registryList) {
										addressListSB.append(item).append(",");
									}
									addressListStr = addressListSB.toString();
									addressListStr = addressListStr.substring(0, addressListStr.length()-1);
								}
								group.setAddressList(addressListStr);
								group.setUpdateTime(new Date());

								XxlJobAdminConfig.getAdminConfig().getXxlJobGroupDao().update(group);
							}
						}
					} catch (Exception e) {
						if (!toStop) {
							logger.error(">>>>>>>>>>> xxl-job, job registry monitor thread error:{}", e);
						}
					}
					try {
						TimeUnit.SECONDS.sleep(RegistryConfig.BEAT_TIMEOUT);
					} catch (InterruptedException e) {
						if (!toStop) {
							logger.error(">>>>>>>>>>> xxl-job, job registry monitor thread error:{}", e);
						}
					}
				}
				logger.info(">>>>>>>>>>> xxl-job, job registry monitor thread stop");
			}
});
```
registryMonitorThread 是 Admin 端负责“定时清理失效执行器注册信息（心跳过期）”的后台监控线程,它负责把已经死掉的executor从调度视野里移除。

它先扫描xxl_job_registry表如果大于90秒物理删除过期的executor，把有效的executor更新到xxl_job_group表中主要更新的是address_list,休眠30秒再重新执行。

## 三、请求Admin端注册的逻辑

执行端执行注册的时候会请求Admin端，执行的Controller是`JobApiController`,执行方法是api方法，该方法源码如下:

```java
 @RequestMapping("/{uri}")
    @ResponseBody
    @PermissionLimit(limit=false)
    public ReturnT<String> api(HttpServletRequest request, @PathVariable("uri") String uri, @RequestBody(required = false) String data) {

        if (!"POST".equalsIgnoreCase(request.getMethod())) {
            return new ReturnT<String>(ReturnT.FAIL_CODE, "invalid request, HttpMethod not support.");
        }
        if (uri==null || uri.trim().length()==0) {
            return new ReturnT<String>(ReturnT.FAIL_CODE, "invalid request, uri-mapping empty.");
        }
        if (XxlJobAdminConfig.getAdminConfig().getAccessToken()!=null
                && XxlJobAdminConfig.getAdminConfig().getAccessToken().trim().length()>0
                && !XxlJobAdminConfig.getAdminConfig().getAccessToken().equals(request.getHeader(XxlJobRemotingUtil.XXL_JOB_ACCESS_TOKEN))) {
            return new ReturnT<String>(ReturnT.FAIL_CODE, "The access token is wrong.");
        }
        if ("callback".equals(uri)) {
            List<HandleCallbackParam> callbackParamList = GsonTool.fromJson(data, List.class, HandleCallbackParam.class);
            return adminBiz.callback(callbackParamList);
        } else if ("registry".equals(uri)) {
            RegistryParam registryParam = GsonTool.fromJson(data, RegistryParam.class);
            return adminBiz.registry(registryParam);
        } else if ("registryRemove".equals(uri)) {
            RegistryParam registryParam = GsonTool.fromJson(data, RegistryParam.class);
            return adminBiz.registryRemove(registryParam);
        } else {
            return new ReturnT<String>(ReturnT.FAIL_CODE, "invalid request, uri-mapping("+ uri +") not found.");
        }

    }
```
注册逻辑是执行的这段代码`adminBiz.registry(registryParam)`最终调用对象JobRegistryHelper的registry方法，该方法源码如下:
```java
public ReturnT<String> registry(RegistryParam registryParam) {

		if (!StringUtils.hasText(registryParam.getRegistryGroup())
				|| !StringUtils.hasText(registryParam.getRegistryKey())
				|| !StringUtils.hasText(registryParam.getRegistryValue())) {
			return new ReturnT<String>(ReturnT.FAIL_CODE, "Illegal Argument.");
		}
		registryOrRemoveThreadPool.execute(new Runnable() {
			@Override
			public void run() {
				int ret = XxlJobAdminConfig.getAdminConfig().getXxlJobRegistryDao().registryUpdate(registryParam.getRegistryGroup(), registryParam.getRegistryKey(), registryParam.getRegistryValue(), new Date());
				if (ret < 1) {
					XxlJobAdminConfig.getAdminConfig().getXxlJobRegistryDao().registrySave(registryParam.getRegistryGroup(), registryParam.getRegistryKey(), registryParam.getRegistryValue(), new Date());

					// fresh
					freshGroupRegistryInfo(registryParam);
				}
			}
		});

		return ReturnT.SUCCESS;
	}
```
主要是通过线程池异步更新xxl_job_registry表中的registry_value字段，即executor的回调地址。

## 四、Admin端调度的逻辑

页面执行调度调用的Controller类是`JobInfoController`执行的方法是triggerJob,该方法内部调用`JobTriggerPoolHelper.trigger(id, TriggerTypeEnum.MANUAL, -1, null, executorParam, addressList);
`又再次调用了`JobTriggerPoolHelper.addTrigger`方法,下面详细看一下这个方法的源码:

```java
public void addTrigger(final int jobId,
                           final TriggerTypeEnum triggerType,
                           final int failRetryCount,
                           final String executorShardingParam,
                           final String executorParam,
                           final String addressList) {

        ThreadPoolExecutor triggerPool_ = fastTriggerPool;
        AtomicInteger jobTimeoutCount = jobTimeoutCountMap.get(jobId);
        if (jobTimeoutCount!=null && jobTimeoutCount.get() > 10) {  
            triggerPool_ = slowTriggerPool;
        }

        triggerPool_.execute(new Runnable() {
            @Override
            public void run() {
                long start = System.currentTimeMillis();
                try {
                    XxlJobTrigger.trigger(jobId, triggerType, failRetryCount, executorShardingParam, executorParam, addressList);
                } catch (Exception e) {
                    logger.error(e.getMessage(), e);
                } finally {

                    long minTim_now = System.currentTimeMillis()/60000;
                    if (minTim != minTim_now) {
                        minTim = minTim_now;
                        jobTimeoutCountMap.clear();
                    }
                    long cost = System.currentTimeMillis()-start;
                    if (cost > 500) { 
                        AtomicInteger timeoutCount = jobTimeoutCountMap.putIfAbsent(jobId, new AtomicInteger(1));
                        if (timeoutCount != null) {
                            timeoutCount.incrementAndGet();
                        }
                    }
                }
            }
        });
    }
```
根据调度jobId去`ConcurrentMap<Integer, AtomicInteger> jobTimeoutCountMap`中查找AtomicInteger的值，这个值代表的是这个 job 最近 1 分钟内触发慢的次数，这个规则是这样的：

- 1 分钟内 触发耗时 > 500ms 超过 10 次

- 认为这是一个问题任务 / 慢任务

- 放到 slowTriggerPool

慢任务用slowTriggerPool线程池执行。

详细看一下run方法执行逻辑,先获取当前时间的毫秒数start,调用`XxlJobTrigger.trigger`方法真正执行调度逻辑，接着这段代码获取当前时间的分钟数`long minTim_now = System.currentTimeMillis()/60000;`,
得到`JobTriggerPoolHelper`创建对象时候成员属性时间值`private volatile long minTim = System.currentTimeMillis()/60000`,如果minTim_now != minTim说明不是在同一分钟时间范围内就把
jobTimeoutCountMap清空重新算AtomicInteger值，这就是一个时间滑动窗口的算法，主要是判断同一个调度一分钟内是否有超过10个的调度，如果有就用慢的线程池执行。

接着得到cost值判断执行调度时间是否大于500毫秒，执行这段代码` AtomicInteger timeoutCount = jobTimeoutCountMap.putIfAbsent(jobId, new AtomicInteger(1))`判断如果之前存在就加1直到大于10次就和第一段
执行代码关联上了。

下面再看一下真正的调度执行逻辑，它调用的是XxlJobTrigger类的trigger静态方法，该方法源码如下:
```java
public static void trigger(int jobId,
                               TriggerTypeEnum triggerType,
                               int failRetryCount,
                               String executorShardingParam,
                               String executorParam,
                               String addressList) {

        XxlJobInfo jobInfo = XxlJobAdminConfig.getAdminConfig().getXxlJobInfoDao().loadById(jobId);
        if (jobInfo == null) {
            logger.warn(">>>>>>>>>>>> trigger fail, jobId invalid，jobId={}", jobId);
            return;
        }
        if (executorParam != null) {
            jobInfo.setExecutorParam(executorParam);
        }
        int finalFailRetryCount = failRetryCount>=0?failRetryCount:jobInfo.getExecutorFailRetryCount();
        XxlJobGroup group = XxlJobAdminConfig.getAdminConfig().getXxlJobGroupDao().load(jobInfo.getJobGroup());

        if (addressList!=null && addressList.trim().length()>0) {
            group.setAddressType(1);
            group.setAddressList(addressList.trim());
        }
        int[] shardingParam = null;
        if (executorShardingParam!=null){
            String[] shardingArr = executorShardingParam.split("/");
            if (shardingArr.length==2 && isNumeric(shardingArr[0]) && isNumeric(shardingArr[1])) {
                shardingParam = new int[2];
                shardingParam[0] = Integer.valueOf(shardingArr[0]);
                shardingParam[1] = Integer.valueOf(shardingArr[1]);
            }
        }
        if (ExecutorRouteStrategyEnum.SHARDING_BROADCAST==ExecutorRouteStrategyEnum.match(jobInfo.getExecutorRouteStrategy(), null)
                && group.getRegistryList()!=null && !group.getRegistryList().isEmpty()
                && shardingParam==null) {
            for (int i = 0; i < group.getRegistryList().size(); i++) {
                processTrigger(group, jobInfo, finalFailRetryCount, triggerType, i, group.getRegistryList().size());
            }
        } else {
            if (shardingParam == null) {
                shardingParam = new int[]{0, 1};
            }
            processTrigger(group, jobInfo, finalFailRetryCount, triggerType, shardingParam[0], shardingParam[1]);
        }
    }
```
查出XxlJobInfo对象，如果调度页面传递的参数executorParam不为空设置到XxlJobInfo对象上,获取页面配置的重试次数finalFailRetryCount,根据
jobInfo查出XxlJobGroup对象获取有效的executor的回调执行地址，获取页面配置的路由策略如果是分片广播，循环回调地址然后调用processTrigger方法，该方法源码如下:
```java
private static void processTrigger(XxlJobGroup group, XxlJobInfo jobInfo, int finalFailRetryCount, TriggerTypeEnum triggerType, int index, int total){

        ExecutorBlockStrategyEnum blockStrategy = ExecutorBlockStrategyEnum.match(jobInfo.getExecutorBlockStrategy(), ExecutorBlockStrategyEnum.SERIAL_EXECUTION);  // block strategy
        ExecutorRouteStrategyEnum executorRouteStrategyEnum = ExecutorRouteStrategyEnum.match(jobInfo.getExecutorRouteStrategy(), null);    // route strategy
        String shardingParam = (ExecutorRouteStrategyEnum.SHARDING_BROADCAST==executorRouteStrategyEnum)?String.valueOf(index).concat("/").concat(String.valueOf(total)):null;

        XxlJobLog jobLog = new XxlJobLog();
        jobLog.setJobGroup(jobInfo.getJobGroup());
        jobLog.setJobId(jobInfo.getId());
        jobLog.setTriggerTime(new Date());
        XxlJobAdminConfig.getAdminConfig().getXxlJobLogDao().save(jobLog);
        logger.debug(">>>>>>>>>>> xxl-job trigger start, jobId:{}", jobLog.getId());

        TriggerParam triggerParam = new TriggerParam();
        triggerParam.setJobId(jobInfo.getId());
        triggerParam.setExecutorHandler(jobInfo.getExecutorHandler());
        triggerParam.setExecutorParams(jobInfo.getExecutorParam());
        triggerParam.setExecutorBlockStrategy(jobInfo.getExecutorBlockStrategy());
        triggerParam.setExecutorTimeout(jobInfo.getExecutorTimeout());
        triggerParam.setLogId(jobLog.getId());
        triggerParam.setLogDateTime(jobLog.getTriggerTime().getTime());
        triggerParam.setGlueType(jobInfo.getGlueType());
        triggerParam.setGlueSource(jobInfo.getGlueSource());
        triggerParam.setGlueUpdatetime(jobInfo.getGlueUpdatetime().getTime());
        triggerParam.setBroadcastIndex(index);
        triggerParam.setBroadcastTotal(total);

        String address = null;
        ReturnT<String> routeAddressResult = null;
        if (group.getRegistryList()!=null && !group.getRegistryList().isEmpty()) {
            if (ExecutorRouteStrategyEnum.SHARDING_BROADCAST == executorRouteStrategyEnum) {
                if (index < group.getRegistryList().size()) {
                    address = group.getRegistryList().get(index);
                } else {
                    address = group.getRegistryList().get(0);
                }
            } else {
                routeAddressResult = executorRouteStrategyEnum.getRouter().route(triggerParam, group.getRegistryList());
                if (routeAddressResult.getCode() == ReturnT.SUCCESS_CODE) {
                    address = routeAddressResult.getContent();
                }
            }
        } else {
            routeAddressResult = new ReturnT<String>(ReturnT.FAIL_CODE, I18nUtil.getString("jobconf_trigger_address_empty"));
        }

        ReturnT<String> triggerResult = null;
        if (address != null) {
            triggerResult = runExecutor(triggerParam, address);
        } else {
            triggerResult = new ReturnT<String>(ReturnT.FAIL_CODE, null);
        }
        StringBuffer triggerMsgSb = new StringBuffer();
        triggerMsgSb.append(I18nUtil.getString("jobconf_trigger_type")).append("：").append(triggerType.getTitle());
        triggerMsgSb.append("<br>").append(I18nUtil.getString("jobconf_trigger_admin_adress")).append("：").append(IpUtil.getIp());
        triggerMsgSb.append("<br>").append(I18nUtil.getString("jobconf_trigger_exe_regtype")).append("：")
                .append( (group.getAddressType() == 0)?I18nUtil.getString("jobgroup_field_addressType_0"):I18nUtil.getString("jobgroup_field_addressType_1") );
        triggerMsgSb.append("<br>").append(I18nUtil.getString("jobconf_trigger_exe_regaddress")).append("：").append(group.getRegistryList());
        triggerMsgSb.append("<br>").append(I18nUtil.getString("jobinfo_field_executorRouteStrategy")).append("：").append(executorRouteStrategyEnum.getTitle());
        if (shardingParam != null) {
            triggerMsgSb.append("("+shardingParam+")");
        }
        triggerMsgSb.append("<br>").append(I18nUtil.getString("jobinfo_field_executorBlockStrategy")).append("：").append(blockStrategy.getTitle());
        triggerMsgSb.append("<br>").append(I18nUtil.getString("jobinfo_field_timeout")).append("：").append(jobInfo.getExecutorTimeout());
        triggerMsgSb.append("<br>").append(I18nUtil.getString("jobinfo_field_executorFailRetryCount")).append("：").append(finalFailRetryCount);

        triggerMsgSb.append("<br><br><span style=\"color:#00c0ef;\" > >>>>>>>>>>>"+ I18nUtil.getString("jobconf_trigger_run") +"<<<<<<<<<<< </span><br>")
                .append((routeAddressResult!=null&&routeAddressResult.getMsg()!=null)?routeAddressResult.getMsg()+"<br><br>":"").append(triggerResult.getMsg()!=null?triggerResult.getMsg():"");

        jobLog.setExecutorAddress(address);
        jobLog.setExecutorHandler(jobInfo.getExecutorHandler());
        jobLog.setExecutorParam(jobInfo.getExecutorParam());
        jobLog.setExecutorShardingParam(shardingParam);
        jobLog.setExecutorFailRetryCount(finalFailRetryCount);
        jobLog.setTriggerCode(triggerResult.getCode());
        jobLog.setTriggerMsg(triggerMsgSb.toString());
        XxlJobAdminConfig.getAdminConfig().getXxlJobLogDao().updateTriggerInfo(jobLog);
        logger.debug(">>>>>>>>>>> xxl-job trigger end, jobId:{}", jobLog.getId());
    }
```
获取页面配置阻塞处理策略,然后如果路由策略是分片广播拼接shardingParam，格式是当前调度的url的索引下标/调度总个数,保存xxl_job_log调度日志表，主要字段有job_id、调度时间、job_group等,创建TriggerParam对象，封装一些调度器参数
例如job_id、JobHandler名称也就是我们的业务方法、执行参数executorParam这是页面指定的、阻塞处理策略executorBlockStrategy、任务超时时间executorTimeout、日志id、
运行模式glueType一般我们都指定BEAN、当前分片广播索引broadcastIndex、分片广播总数broadcastTotal。

接着调用runExecutor方法传入TriggerParam、executor调度的回调地址address，该方法源码如下:
```java
public static ReturnT<String> runExecutor(TriggerParam triggerParam, String address){
        ReturnT<String> runResult = null;
        try {
            ExecutorBiz executorBiz = XxlJobScheduler.getExecutorBiz(address);
            runResult = executorBiz.run(triggerParam);
        } catch (Exception e) {
            logger.error(">>>>>>>>>>> xxl-job trigger error, please check if the executor[{}] is running.", address, e);
            runResult = new ReturnT<String>(ReturnT.FAIL_CODE, ThrowableUtil.toString(e));
        }

        StringBuffer runResultSB = new StringBuffer(I18nUtil.getString("jobconf_trigger_run") + "：");
        runResultSB.append("<br>address：").append(address);
        runResultSB.append("<br>code：").append(runResult.getCode());
        runResultSB.append("<br>msg：").append(runResult.getMsg());

        runResult.setMsg(runResultSB.toString());
        return runResult;
}
```
调用`XxlJobScheduler.getExecutorBiz(address)`方法给每一个address实例化一个ExecutorBizClient对象，它里面包含执行器回调地址和token信息，接着调用run方法进行Http请求，请求的rul地址
是`http:xxxx/run`,封装返回结果为ReturnT对象，然后更新xxl_job_log日志表的执行结果。

## 五、执行器返回的逻辑

执行的分支逻辑是这段代码逻辑:
```java
if ("callback".equals(uri)) {
        List<HandleCallbackParam> callbackParamList = GsonTool.fromJson(data, List.class, HandleCallbackParam.class);
        return adminBiz.callback(callbackParamList);
} 
```
最终调用的是JobCompleteHelper对象的callback方法，该方法源码如下:
```java
public ReturnT<String> callback(List<HandleCallbackParam> callbackParamList) {

		callbackThreadPool.execute(new Runnable() {
			@Override
			public void run() {
				for (HandleCallbackParam handleCallbackParam: callbackParamList) {
					ReturnT<String> callbackResult = callback(handleCallbackParam);
					logger.debug(">>>>>>>>> JobApiController.callback {}, handleCallbackParam={}, callbackResult={}",
							(callbackResult.getCode()== ReturnT.SUCCESS_CODE?"success":"fail"), handleCallbackParam, callbackResult);
				}
			}
		});

		return ReturnT.SUCCESS;
}
```
用callbackThreadPool线程池异步执行回调逻辑，调用callback方法，该方法源码如下:
```java
private ReturnT<String> callback(HandleCallbackParam handleCallbackParam) {
    XxlJobLog log = XxlJobAdminConfig.getAdminConfig().getXxlJobLogDao().load(handleCallbackParam.getLogId());
    if (log == null) {
        return new ReturnT<String>(ReturnT.FAIL_CODE, "log item not found.");
    }
    if (log.getHandleCode() > 0) {
        return new ReturnT<String>(ReturnT.FAIL_CODE, "log repeate callback.");
    }
    StringBuffer handleMsg = new StringBuffer();
    if (log.getHandleMsg()!=null) {
        handleMsg.append(log.getHandleMsg()).append("<br>");
    }
    if (handleCallbackParam.getHandleMsg() != null) {
        handleMsg.append(handleCallbackParam.getHandleMsg());
    }
    log.setHandleTime(new Date());
    log.setHandleCode(handleCallbackParam.getHandleCode());
    log.setHandleMsg(handleMsg.toString());
    XxlJobCompleter.updateHandleInfoAndFinish(log);
    return ReturnT.SUCCESS;
}
```
查询出XxlJobLog对象设置执行任务返回的handleMsg、handleCode，然后调用`XxlJobCompleter.updateHandleInfoAndFinish`方法，该方法源码如下:
```java
    public static int updateHandleInfoAndFinish(XxlJobLog xxlJobLog) {

        finishJob(xxlJobLog);

        if (xxlJobLog.getHandleMsg().length() > 15000) {
            xxlJobLog.setHandleMsg( xxlJobLog.getHandleMsg().substring(0, 15000) );
        }
        return XxlJobAdminConfig.getAdminConfig().getXxlJobLogDao().updateHandleInfo(xxlJobLog);
    }
```
首先调用的是finishJob方法，该方法源码如下:
```java
private static void finishJob(XxlJobLog xxlJobLog){
     String triggerChildMsg = null;
        if (XxlJobContext.HANDLE_COCE_SUCCESS == xxlJobLog.getHandleCode()) {
            XxlJobInfo xxlJobInfo = XxlJobAdminConfig.getAdminConfig().getXxlJobInfoDao().loadById(xxlJobLog.getJobId());
            if (xxlJobInfo!=null && xxlJobInfo.getChildJobId()!=null && xxlJobInfo.getChildJobId().trim().length()>0) {
                triggerChildMsg = "<br><br><span style=\"color:#00c0ef;\" > >>>>>>>>>>>"+ I18nUtil.getString("jobconf_trigger_child_run") +"<<<<<<<<<<< </span><br>";

                String[] childJobIds = xxlJobInfo.getChildJobId().split(",");
                for (int i = 0; i < childJobIds.length; i++) {
                    int childJobId = (childJobIds[i]!=null && childJobIds[i].trim().length()>0 && isNumeric(childJobIds[i]))?Integer.valueOf(childJobIds[i]):-1;
                    if (childJobId > 0) {

                        JobTriggerPoolHelper.trigger(childJobId, TriggerTypeEnum.PARENT, -1, null, null, null);
                        ReturnT<String> triggerChildResult = ReturnT.SUCCESS;

                        // add msg
                        triggerChildMsg += MessageFormat.format(I18nUtil.getString("jobconf_callback_child_msg1"),
                                (i+1),
                                childJobIds.length,
                                childJobIds[i],
                                (triggerChildResult.getCode()==ReturnT.SUCCESS_CODE?I18nUtil.getString("system_success"):I18nUtil.getString("system_fail")),
                                triggerChildResult.getMsg());
                    } else {
                        triggerChildMsg += MessageFormat.format(I18nUtil.getString("jobconf_callback_child_msg2"),
                                (i+1),
                                childJobIds.length,
                                childJobIds[i]);
                    }
                }

            }
        }
        if (triggerChildMsg != null) {
            xxlJobLog.setHandleMsg( xxlJobLog.getHandleMsg() + triggerChildMsg );
        }
    }
```
这里判断页面是否配置了子任务如果配置了调度子任务，需要注意一下此处不等待子任务返回结果只是增加一些日志说我调度了子任务。

执行完成更新日志信息表xxl_job_log。

## 六、调度算法源码分析

在执行调度时候还有个逻辑就是如果当前页面配置路由策略不是分片广播会根据配置的选择相应的路由策略算法，页面可以配置的有十种，截图如下:

![xxl-job2.png](..%2Fimg%2Fxxl-job2.png)

源码中会根据页面配置的选择对应的类进行路由，源码是这样的:
```java
    ExecutorRouteStrategyEnum executorRouteStrategyEnum = ExecutorRouteStrategyEnum.match(jobInfo.getExecutorRouteStrategy(), null);
    routeAddressResult = executorRouteStrategyEnum.getRouter().route(triggerParam, group.getRegistryList());
```
先看路由策略中的**第一个**算法，它的源码比较简单就是取registryList的第一个代码如下:
```java
  public ReturnT<String> route(TriggerParam triggerParam, List<String> addressList){
        return new ReturnT<String>(addressList.get(0));
 }
```
路由策略中的**最后一个**算法，它的源码也比较简单就是取registryList的最后一个代码如下:
```java
    public ReturnT<String> route(TriggerParam triggerParam, List<String> addressList) {
        return new ReturnT<String>(addressList.get(addressList.size()-1));
    }
```
再来看一下路由策略中的**轮询**算法，它的源码如下:
```java
    public ReturnT<String> route(TriggerParam triggerParam, List<String> addressList) {
        String address = addressList.get(count(triggerParam.getJobId())%addressList.size());
        return new ReturnT<String>(address);
    }
```
又调用了count方法获取计数，它的源码如下：
```java
private static int count(int jobId) {
        // cache clear
        if (System.currentTimeMillis() > CACHE_VALID_TIME) {
            routeCountEachJob.clear();
            CACHE_VALID_TIME = System.currentTimeMillis() + 1000*60*60*24;
        }

        AtomicInteger count = routeCountEachJob.get(jobId);
        if (count == null || count.get() > 1000000) {
            // 初始化时主动Random一次，缓解首次压力
            count = new AtomicInteger(new Random().nextInt(100));
        } else {
            // count++
            count.addAndGet(1);
        }
        routeCountEachJob.put(jobId, count);
        return count.get();
}
```
判断`System.currentTimeMillis() > CACHE_VALID_TIME`条件是否成立，这段代码的主要作用如下:

- 防止 routeCountEachJob 无限增长
- 防止单个 job 的 count 变成超级大数
- 每天给路由一个“自然重置点”

然后取出 jobId 对应的计数器,每个 jobId 都有自己独立的路由序列不会相互干扰，执行这段代码是一个精华:
```java
if (count == null || count.get() > 1000000) {
    count = new AtomicInteger(new Random().nextInt(100));
}

```
就是刚启动第一次访问或者计数大于10万不是从0开始而是选择一个随机数这样做是防止第一次路由时候都访问到第一台机器上，随机的好处是初始路由位置被打散，多个 job 同时启动时executor 负载更均匀。

正常情况下是count.addAndGet(1),然后放回到map中最后返回计数。

然后计数以后对addressList.size()取模这样就可以达到轮询的效果。

再来看一下路由策略中的**随机**算法，它的源码如下:
```java
    @Override
    public ReturnT<String> route(TriggerParam triggerParam, List<String> addressList) {
        String address = addressList.get(localRandom.nextInt(addressList.size()));
        return new ReturnT<String>(address);
    }
```
这种算法比较简单就是取小于addressList.size()的随机下标值。

再来看一下路由策略中的**故障转移**算法，它的源码如下:
```java
public ReturnT<String> route(TriggerParam triggerParam, List<String> addressList) {

        StringBuffer beatResultSB = new StringBuffer();
        for (String address : addressList) {
            ReturnT<String> beatResult = null;
            try {
                ExecutorBiz executorBiz = XxlJobScheduler.getExecutorBiz(address);
                beatResult = executorBiz.beat();
            } catch (Exception e) {
                logger.error(e.getMessage(), e);
                beatResult = new ReturnT<String>(ReturnT.FAIL_CODE, ""+e );
            }
            beatResultSB.append( (beatResultSB.length()>0)?"<br><br>":"")
                    .append(I18nUtil.getString("jobconf_beat") + "：")
                    .append("<br>address：").append(address)
                    .append("<br>code：").append(beatResult.getCode())
                    .append("<br>msg：").append(beatResult.getMsg());

            if (beatResult.getCode() == ReturnT.SUCCESS_CODE) {

                beatResult.setMsg(beatResultSB.toString());
                beatResult.setContent(address);
                return beatResult;
            }
        }
        return new ReturnT<String>(ReturnT.FAIL_CODE, beatResultSB.toString());

}
```
循环遍历所有执行端地址，调用`executorBiz.beat()`方法试探一下执行端是否可以连通，如果是连通状态返回一个成功的状态，如果连不同就返回失败的ReturnT对象或者发生异常走到
catch块得到一个错误的ReturnT对象，不管成功与否都把结果添加到StringBuffer中，如果是连通就用当前这个执行器地址，否则就继续下一次循环如果依然没有连通的执行器
就返回异常ReturnT对象。

再来看一下路由策略中的**忙碌转移**算法，它的源码如下:
```java
public ReturnT<String> route(TriggerParam triggerParam, List<String> addressList) {
        StringBuffer idleBeatResultSB = new StringBuffer();
        for (String address : addressList) {
            // beat
            ReturnT<String> idleBeatResult = null;
            try {
                ExecutorBiz executorBiz = XxlJobScheduler.getExecutorBiz(address);
                idleBeatResult = executorBiz.idleBeat(new IdleBeatParam(triggerParam.getJobId()));
            } catch (Exception e) {
                logger.error(e.getMessage(), e);
                idleBeatResult = new ReturnT<String>(ReturnT.FAIL_CODE, ""+e );
            }
            idleBeatResultSB.append( (idleBeatResultSB.length()>0)?"<br><br>":"")
                    .append(I18nUtil.getString("jobconf_idleBeat") + "：")
                    .append("<br>address：").append(address)
                    .append("<br>code：").append(idleBeatResult.getCode())
                    .append("<br>msg：").append(idleBeatResult.getMsg());

            // beat success
            if (idleBeatResult.getCode() == ReturnT.SUCCESS_CODE) {
                idleBeatResult.setMsg(idleBeatResultSB.toString());
                idleBeatResult.setContent(address);
                return idleBeatResult;
            }
        }

        return new ReturnT<String>(ReturnT.FAIL_CODE, idleBeatResultSB.toString());
    }
```
循环遍历所有执行端地址，调用`executorBiz.idleBeat()`方法试探一下执行端的线程是否有执行的任务，如果有就返回错误的ReturnT对象，如果没有就返回
正常的ReturnT对象，Admin端根据返回的Code码是否是成功，如果有一个是成功的就返回当前执行端地址。

再来看一下路由策略中的**最近最久未使用**算法，它的源码如下:
```java
public class ExecutorRouteLRU extends ExecutorRouter {

    private static ConcurrentMap<Integer, LinkedHashMap<String, String>> jobLRUMap = new ConcurrentHashMap<Integer, LinkedHashMap<String, String>>();
    private static long CACHE_VALID_TIME = 0;

    public String route(int jobId, List<String> addressList) {

        // cache clear
        if (System.currentTimeMillis() > CACHE_VALID_TIME) {
            jobLRUMap.clear();
            CACHE_VALID_TIME = System.currentTimeMillis() + 1000*60*60*24;
        }

        // init lru
        LinkedHashMap<String, String> lruItem = jobLRUMap.get(jobId);
        if (lruItem == null) {
            /**
             * LinkedHashMap
             *      a、accessOrder：true=访问顺序排序（get/put时排序）；false=插入顺序排期；
             *      b、removeEldestEntry：新增元素时将会调用，返回true时会删除最老元素；可封装LinkedHashMap并重写该方法，比如定义最大容量，超出是返回true即可实现固定长度的LRU算法；
             */
            lruItem = new LinkedHashMap<String, String>(16, 0.75f, true);
            jobLRUMap.putIfAbsent(jobId, lruItem);
        }

        // put new
        for (String address: addressList) {
            if (!lruItem.containsKey(address)) {
                lruItem.put(address, address);
            }
        }
        // remove old
        List<String> delKeys = new ArrayList<>();
        for (String existKey: lruItem.keySet()) {
            if (!addressList.contains(existKey)) {
                delKeys.add(existKey);
            }
        }
        if (delKeys.size() > 0) {
            for (String delKey: delKeys) {
                lruItem.remove(delKey);
            }
        }

        // load
        String eldestKey = lruItem.entrySet().iterator().next().getKey();
        String eldestValue = lruItem.get(eldestKey);
        return eldestValue;
    }
```
每 24 小时清空所有 job 的 LRU 记录,防止job已删除、内存无限增长，在创建LinkedHashMap时候指定`accessOrder = true`表示按照按访问顺序排序,get/put时候
会把该 key 移到链表尾部，这样的话头部就是最久没用的（LRU）尾部是最近用过的。

执行put操作:如果有 新执行器上线加入 LRU 列表初始位置在尾部。

执行删除操作:Admin 发现执行器列表变化,LRU 中 剔除已下线地址,保证 LRU 列表是当前真实可用执行器。

接下来下面这三行代码是最主要的：
```java
String eldestKey = lruItem.entrySet().iterator().next().getKey();
String eldestValue = lruItem.get(eldestKey);
return eldestValue;
```
iterator().next(),拿的是 LinkedHashMap 的第一个元素,在 accessOrder = true 情况下：最久没被访问的节点。

再来看一下路由策略中的**最不经常使用**算法，它的源码如下:
```java
public String route(int jobId, List<String> addressList) {

        // cache clear
        if (System.currentTimeMillis() > CACHE_VALID_TIME) {
            jobLfuMap.clear();
            CACHE_VALID_TIME = System.currentTimeMillis() + 1000*60*60*24;
        }

        // lfu item init
        HashMap<String, Integer> lfuItemMap = jobLfuMap.get(jobId);     // Key排序可以用TreeMap+构造入参Compare；Value排序暂时只能通过ArrayList；
        if (lfuItemMap == null) {
            lfuItemMap = new HashMap<String, Integer>();
            jobLfuMap.putIfAbsent(jobId, lfuItemMap);   // 避免重复覆盖
        }

        // put new
        for (String address: addressList) {
            if (!lfuItemMap.containsKey(address) || lfuItemMap.get(address) >1000000 ) {
                lfuItemMap.put(address, new Random().nextInt(addressList.size()));  // 初始化时主动Random一次，缓解首次压力
            }
        }
        // remove old
        List<String> delKeys = new ArrayList<>();
        for (String existKey: lfuItemMap.keySet()) {
            if (!addressList.contains(existKey)) {
                delKeys.add(existKey);
            }
        }
        if (delKeys.size() > 0) {
            for (String delKey: delKeys) {
                lfuItemMap.remove(delKey);
            }
        }

        // load least userd count address
        List<Map.Entry<String, Integer>> lfuItemList = new ArrayList<Map.Entry<String, Integer>>(lfuItemMap.entrySet());
        Collections.sort(lfuItemList, new Comparator<Map.Entry<String, Integer>>() {
            @Override
            public int compare(Map.Entry<String, Integer> o1, Map.Entry<String, Integer> o2) {
                return o1.getValue().compareTo(o2.getValue());
            }
        });

        Map.Entry<String, Integer> addressItem = lfuItemList.get(0);
        String minAddress = addressItem.getKey();
        addressItem.setValue(addressItem.getValue() + 1);

        return addressItem.getKey();
    }
```
还是每 24 小时清空所有 job 的 LFU 统计，然后每个job一个LFU统计表，如果新执行器上线并且address 是新加入的不会初始化为0而是会随机选取一个值大小不超过执行的总数量，
避免所有新节点同时成为最小值。

同样是循环清理下线的执行器，然后是最核心逻辑是寻找最少使用的执行器，这里把把 Map 转成 List按 count 从小到大排序，然后选取最小的并把最小的次数+1。

再来看一下路由策略中最后一个**一致性HASH**算法，它的源码如下:

````java
 /**
     * get hash code on 2^32 ring (md5散列的方式计算hash值)
     * @param key
     * @return
     */
    private static long hash(String key) {

        // md5 byte
        MessageDigest md5;
        try {
            md5 = MessageDigest.getInstance("MD5");
        } catch (NoSuchAlgorithmException e) {
            throw new RuntimeException("MD5 not supported", e);
        }
        md5.reset();
        byte[] keyBytes = null;
        try {
            keyBytes = key.getBytes("UTF-8");
        } catch (UnsupportedEncodingException e) {
            throw new RuntimeException("Unknown string :" + key, e);
        }

        md5.update(keyBytes);
        byte[] digest = md5.digest();

        // hash code, Truncate to 32-bits
        long hashCode = ((long) (digest[3] & 0xFF) << 24)
                | ((long) (digest[2] & 0xFF) << 16)
                | ((long) (digest[1] & 0xFF) << 8)
                | (digest[0] & 0xFF);

        long truncateHashCode = hashCode & 0xffffffffL;
        return truncateHashCode;
    }

    public String hashJob(int jobId, List<String> addressList) {

        // ------A1------A2-------A3------
        // -----------J1------------------
        TreeMap<Long, String> addressRing = new TreeMap<Long, String>();
        for (String address: addressList) {
            for (int i = 0; i < VIRTUAL_NODE_NUM; i++) {
                long addressHash = hash("SHARD-" + address + "-NODE-" + i);
                addressRing.put(addressHash, address);
            }
        }

        long jobHash = hash(String.valueOf(jobId));
        SortedMap<Long, String> lastRing = addressRing.tailMap(jobHash);
        if (!lastRing.isEmpty()) {
            return lastRing.get(lastRing.firstKey());
        }
        return addressRing.firstEntry().getValue();
    }

    @Override
    public ReturnT<String> route(TriggerParam triggerParam, List<String> addressList) {
        String address = hashJob(triggerParam.getJobId(), addressList);
        return new ReturnT<String>(address);
    }
````
首先用 MD5 对字符串（jobId 或执行器 + 虚拟节点）做散列,将 key 映射到 0~2^32-1 的环上，hash(key) = key 在圆环上的位置。

对每台执行器（address），生成 VIRTUAL_NODE_NUM = 100 个虚拟节点。

虚拟节点名称 "SHARD-" + address + "-NODE-" + i，保证 hash 均匀分布,TreeMap<Long, String> 存储 hash值 → 执行器地址,使用
TreeMap可以保证hash值的顺序性。

jobId 同样通过 hash 映射到 0~2^32-1 环上，这个值就是 job 在环上的位置。

tailMap(jobHash) → 返回 hash ≥ jobHash 的所有节点（顺时针方向上的点），lastRing.firstKey() → 找到 顺时针最近的节点，如果 tailMap 为空 → jobHash 在环的尾端 → 绕回第一个节点。
返回的 value = 执行器地址。这就是一致性 Hash 的核心：顺时针第一个节点负责该 jobId。

如果大家对应一致性HASH算法不了解的可以参考一下这篇文章:
https://developer.huawei.com/consumer/cn/forum/topic/0203810951415790

## 七、结束语

通过对 XXL-JOB Admin 端源码的分析，我们可以清晰地看到：

一个工业级的调度中心，绝不仅仅是“定时发请求”那么简单。

注册保活层：通过 registryMonitorThread + 心跳过期清理，实现了执行器的自动发现与故障摘除；

调度执行层：通过 滑动窗口算法 动态识别慢任务，并将其隔离到独立的 slowTriggerPool，避免了单任务拖垮整个调度系统；

路由策略层：从轮询、随机、故障转移、忙碌转移，到 LRU、LFU、一致性 Hash，覆盖了绝大部分分布式负载均衡场景；

任务链路层：从 trigger 到 runExecutor，再到 callback 闭环，完整支撑了任务调度、执行、回调、子任务触发的全生命周期。



