---
layout: post
title: NiFi之web-api.war启动(4)
categories: nifi
description:
keywords: nifi, web-api, war
---
在上篇讲到，jettyserver类初始化的时候，共加载了9个war包：
![jetty-9war](/images/nifi/jetty-9war.png.png)

这次分析一下，web-api.war的启动流程。我们知道，java web的启动是从web.xml开始引导的，
```xml
	<listener>
		<listener-class>org.springframework.web.context.ContextLoaderListener</listener-class>
	</listener>
	<listener>
		<listener-class>org.apache.nifi.web.contextlistener.ApplicationStartupContextListener</listener-class>
	</listener>
```
&emsp;&emsp;文件中主要观察这两个配置，contextLoaderListener是spring加载自己的context的，不需要关注，主要关注ApplicationStartupContextListener，这是真正的启动代码。
其实在初始化的时候干的事情不多：

1. 调用FlowService的启动和初始化方法。

1. 调用FlowController的onFlowInitialized方法。

## 一、FlowService的初始化 ##

1. 启动一个每500毫秒执行一次的SaveReportingTask线程。
```java
	final SaveHolder holder = StandardFlowService.this.saveHolder.get();
		if (holder == null) {
			return;
		}

		if (logger.isTraceEnabled()) {
			logger.trace("Save request time {} // Current time {}", holder.saveTime.getTime(), new Date());
		}

		final Calendar now = Calendar.getInstance();
		if (holder.saveTime.before(now)) {
			if (logger.isTraceEnabled()) {
				logger.trace("Waiting for write lock and then will save");
			}
			writeLock.lock();
			try {
				dao.save(controller, holder.shouldArchive);
				// Nulling it out if it is still set to our current SaveHolder.  Otherwise leave it alone because it means
				// another save is already pending.
				final boolean noSavePending = StandardFlowService.this.saveHolder.compareAndSet(holder, null);
				logger.info("Saved flow controller {} // Another save pending = {}", controller, !noSavePending);
			} finally {
				writeLock.unlock();
			}
	}
```
&emsp;&emsp;大致意思就是取得用户设置的保存时间，判断当前时间是否超过这个时间，如果超过了这个时间，调用FlowConfigurationDAO的save方法，该类其实是对界面上配置的processors的配置进行保存，例如坐标、连线关系等。
```java
	configFile = flowXmlPath;
	tempFile = configFile.getParent().resolve(configFile.toFile().getName() + ".new.xml.gz");

	try (final OutputStream fileOut = Files.newOutputStream(tempFile);
			final OutputStream outStream = new GZIPOutputStream(fileOut)) {

		final StandardFlowSerializer xmlTransformer = new StandardFlowSerializer(encryptor);
		controller.serialize(xmlTransformer, outStream);

		Files.deleteIfExists(configFile);
		FileUtils.renameFile(tempFile.toFile(), configFile.toFile(), 5, true);
	} catch (final FlowSerializationException fse) {
		throw new IOException(fse);
	} finally {
		Files.deleteIfExists(tempFile);
	}
```
&emsp;&emsp;具体的存储结构不在本文讲述，上面代码的意思就是将界面上的连线等配置保存为一个.xml.gz为扩展名的文件，保存的路径为./config路径下完整名为flow.xml.gz


1. 从保存的flow.xml.gz中读取配置内容到内存，还原回配置，为后续界面展示提供支持，具体的存储和读取细节单独写一篇。

## 二、启动组件 ##

FlowService初始化完成之后根据配置文件中的nifi.flowcontroller.autoResumeState配置项判断需要启动的组件。
```java
	if (startDelayedComponents) {
		LOG.info("Starting {} processors/ports/funnels", startConnectablesAfterInitialization.size() + startRemoteGroupPortsAfterInitialization.size());
		for (final Connectable connectable : startConnectablesAfterInitialization) {
			if (connectable.getScheduledState() == ScheduledState.DISABLED) {
				continue;
			}

			try {
				if (connectable instanceof ProcessorNode) {
					connectable.getProcessGroup().startProcessor((ProcessorNode) connectable);
				} else {
					startConnectable(connectable);
				}
			} catch (final Throwable t) {
				LOG.error("Unable to start {} due to {}", new Object[]{connectable, t.toString()});
				if (LOG.isDebugEnabled()) {
					LOG.error("", t);
				}
			}
		}

		startConnectablesAfterInitialization.clear();

		int startedTransmitting = 0;
		for (final RemoteGroupPort remoteGroupPort : startRemoteGroupPortsAfterInitialization) {
			try {
				remoteGroupPort.getRemoteProcessGroup().startTransmitting(remoteGroupPort);
				startedTransmitting++;
			} catch (final Throwable t) {
				LOG.error("Unable to start transmitting with {} due to {}", new Object[]{remoteGroupPort, t});
			}
		}

		LOG.info("Started {} Remote Group Ports transmitting", startedTransmitting);
		startRemoteGroupPortsAfterInitialization.clear();
	} else {
		// We don't want to start all of the delayed components. However, funnels need to be started anyway
		// because we don't provide users the ability to start or stop them - they are just notional.
		for (final Connectable connectable : startConnectablesAfterInitialization) {
			try {
				if (connectable instanceof Funnel) {
					startConnectable(connectable);
				}
			} catch (final Throwable t) {
				LOG.error("Unable to start {} due to {}", new Object[]{connectable, t});
			}
		}

		startConnectablesAfterInitialization.clear();
		startRemoteGroupPortsAfterInitialization.clear();
	}
```
&emsp;&emsp;当配置为true时候走第一个分之，否则走else分支。代码中可以看到，else分支就是只去启动了中间的inport、outport、remoteport这些东西，不做具体讲述。当配置为true的时候，是遍历了流程里的每个组件并且启动起来，这里看到一部分很有意思的代码:
```java
public <T extends ProcessContext & ControllerServiceLookup> void start(final ScheduledExecutorService taskScheduler,
		final long administrativeYieldMillis, final T processContext, final SchedulingAgentCallback schedulingAgentCallback) {
	if (!this.isValid()) {
		throw new IllegalStateException( "Processor " + this.getName() + " is not in a valid state due to " + this.getValidationErrors());
	}
	final Processor processor = processorRef.get().getProcessor();
	final ComponentLog procLog = new SimpleProcessLogger(StandardProcessorNode.this.getIdentifier(), processor);

	final boolean starting;
	synchronized (this) {
		starting = this.scheduledState.compareAndSet(ScheduledState.STOPPED, ScheduledState.STARTING);
	}

	if (starting) { // will ensure that the Processor represented by this node can only be started once
		final Runnable startProcRunnable = new Runnable() {
			@Override
			public void run() {
				try {
					invokeTaskAsCancelableFuture(schedulingAgentCallback, new Callable<Void>() {
						@Override
						public Void call() throws Exception {
							try (final NarCloseable nc = NarCloseable.withComponentNarLoader(processor.getClass(), processor.getIdentifier())) {
								ReflectionUtils.invokeMethodsWithAnnotation(OnScheduled.class, processor, processContext);
								return null;
							}
						}
					});

					if (scheduledState.compareAndSet(ScheduledState.STARTING, ScheduledState.RUNNING)) {
						schedulingAgentCallback.trigger(); // callback provided by StandardProcessScheduler to essentially initiate component's onTrigger() cycle
					} else { // can only happen if stopProcessor was called before service was transitioned to RUNNING state
						try (final NarCloseable nc = NarCloseable.withComponentNarLoader(processor.getClass(), processor.getIdentifier())) {
							ReflectionUtils.quietlyInvokeMethodsWithAnnotation(OnUnscheduled.class, processor, processContext);
						}
						scheduledState.set(ScheduledState.STOPPED);
					}
				} catch (final Exception e) {
					final Throwable cause = e instanceof InvocationTargetException ? e.getCause() : e;
					procLog.error("{} failed to invoke @OnScheduled method due to {}; processor will not be scheduled to run for {} seconds",
							new Object[]{StandardProcessorNode.this.getProcessor(), cause, administrativeYieldMillis / 1000L}, cause);
					LOG.error("Failed to invoke @OnScheduled method due to {}", cause.toString(), cause);

					ReflectionUtils.quietlyInvokeMethodsWithAnnotation(OnUnscheduled.class, processor, processContext);
					ReflectionUtils.quietlyInvokeMethodsWithAnnotation(OnStopped.class, processor, processContext);

					if (scheduledState.get() != ScheduledState.STOPPING) { // make sure we only continue retry loop if STOP action wasn't initiated
						taskScheduler.schedule(this, administrativeYieldMillis, TimeUnit.MILLISECONDS);
					} else {
						scheduledState.set(ScheduledState.STOPPED);
					}
				}
			}
		};
		taskScheduler.execute(startProcRunnable);
	} else {
		final String procName = processorRef.getClass().getSimpleName();
		LOG.warn("Can not start '" + procName
				+ "' since it's already in the process of being started or it is DISABLED - "
				+ scheduledState.get());
		procLog.warn("Can not start '" + procName
				+ "' since it's already in the process of being started or it is DISABLED - "
				+ scheduledState.get());
	}
}
```
1. 上面的代码是启动一个具体的processor的代码
 
1. 代码涉及到了启动过程中对启动状态的严格判断，从if(starting）这个判断就已经第一次做了只允许一次启动的判断，当然这不是最重要的。

1. 使用了多线程去启动，不阻塞主线程去扫描下一个processor的调度。

1. 代码中，使用了compareAndSet即CAS原语的状态修改，有兴趣可以关注一下CAS的知识，如果非预期的状态和设置，调用processor的OnUnscheduled方法，将状态置为stoped。

1. 最后如果一切启动正常，调用schedulingAgentCallback.trigger()方法：
```java
	SchedulingAgentCallback callback = new SchedulingAgentCallback() {
		@Override
		public void trigger() {
			getSchedulingAgent(procNode).schedule(procNode, scheduleState);
		}

		@Override
		public Future<?> invokeMonitoringTask(Callable<?> task) {
			scheduleState.incrementActiveThreadCount();
			return componentMonitoringThreadPool.submit(task);
		}

		@Override
		public void postMonitor() {
			scheduleState.decrementActiveThreadCount();
		}
	};
```
&emsp;&emsp;其实是判断了一次是何种类型的调度任务，分为3类，分别是：时间驱动、事件驱动和定时驱动。具体的启动代码很多，再分一个章节讲。


到这里为止，基本的启动流程就完成了。