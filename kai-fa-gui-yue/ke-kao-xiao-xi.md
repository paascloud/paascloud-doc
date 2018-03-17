# 1.前言
## 1.1 目的
1. 为开发测试提供指导性文件
2. 为系统今后的扩展提供参考
3. 解决系统中消息不可达问题

## 1.2 范围和功能
## 1.3 适用读者
1. 需要发送MQ分布式系统的开发人员和测试人员
2. 可靠消息服务的开发人员和测试人员

## 1.4 读者须知

本服务需要提供一个sdk和数据库初始语句创建数据库表,并且对外提供可扫描的domain、mapper、service,使用的技术框架zk + mapper3 + pagehelper + feign(edas) , 使用者(上游系统、下游系统) 只需要在对应的接口上写上响应注解即可实现可靠消息, 如果不熟悉上述框架,可选择对应框架替换,比如redis替换zk,放弃mapper3和pagehelper使用传统的mybatis,使用http接口替换fein(eads)的解决办法,本文不提供替换的解决方案
## 1.5 参考文档

```
https://segmentfault.com/a/1190000011479826
```
# 2 系统概述
本文为分布式系统解决方案,此方案涉及 3 个模块：
1. 上游应用，执行业务并发送指令给可靠消息服务并保留消息副本。
2. 可靠消息服务和 MQ消息组件，协调上下游消息的传递，并确保上下游数据的一致性。
3. 下游应用，监听 MQ 的消息并执行自身业务并保留消息副本。

## 2.1业务流程图

![这里写图片描述](http://img.paascloud.net/paascloud/doc/message-2.1.png)

## 2.2数据库表设计

### 2.2.1 可靠消息表

![](http://img.paascloud.net/paascloud/doc/message-db-2.1.1.png)

### 2.2.2 消费者确认表

![](http://img.paascloud.net/paascloud/doc/message-db-2.1.2.png)

### 2.2.3 消费者表


![](http://img.paascloud.net/paascloud/doc/message-db-2.2.3.png)

###2.2.4 生产者表

![](http://img.paascloud.net/paascloud/doc/message-db-2.2.4.png)

###2.2.5 发布关系表

![](http://img.paascloud.net/paascloud/doc/message-db-2.2.5.png)

###2.2.6 消息重发记录表

暂时未设计

###2.2.7 消息订阅关系表

![](http://img.paascloud.net/paascloud/doc/message-db-2.2.7.png)

###2.2.8 消息订阅TAG关系表

![](http://img.paascloud.net/paascloud/doc/message-db-2.2.8.png)

###2.2.9 各个子系统消息落地的消息表

![](http://img.paascloud.net/paascloud/doc/message-db-2.2.9.png)

# 3 详细设计

## 3.1 上游应用执行业务并发送 MQ 消息
![image](http://img.paascloud.net/passcloud/doc/message_3-1-1.png)

上游应用将本地业务执行和消息发送绑定在同一个本地事务中，保证要么本地操作成功并发送 MQ 消息，要么两步操作都失败并回滚。这里可以采用自定义切面完成,后续会有介绍。

![image](http://img.paascloud.net/passcloud/doc/message-3-1-2.png)


1. 上游应用发送待确认消息到可靠消息系统。(本地消息落地)
2. 可靠消息系统保存待确认消息并返回。
3. 上游应用执行本地业务。
4. 上游应用通知可靠消息系统确认业务已执行并发送消息。
5. 可靠消息系统修改消息状态为发送状态并将消息投递到 MQ 中间件。

以上每一步都可能出现失败情况，分析一下这 5 步出现异常后上游业务和消息发送是否一致：


失败步骤 | 现象 | 一致性
---|--- |---
第1步 | 上游应用业务未执行,MQ消息未发送 | 一致
第2步 | 上游应用业务未执行,MQ消息未发送 | 一致
第3步 | 上游应用事物回滚,MQ消息未发送 | 一致
第4步 | 上游应用业务执行,MQ消息未发送 | 不一致
第5步 | 上游应用业务执行,MQ消息未发送 | 不一致

上游应用执行完成，下游应用尚未执行或执行失败时，此事务即处于 BASE 理论的 Soft State 状态。

![image](https://images-cdn.shimo.im/vontc2QyqfIFoRX5/image_3.jpg!thumbnail)




## 3.2 下游应用监听 MQ 消息并执行业务

 1. 下游应用监听 MQ 消息并执行业务，并且将消息的消费结果通知可靠消息服务。(本地消息落地)
 2.  可靠消息的状态需要和下游应用的业务执行保持一致，可靠消息状态不是已完成时，确保下游应用未执行，可靠消息状态是已完成时，确保下游应用已执行。
下游应用和可靠消息服务之间的交互图如下：
![这里写图片描述](http://img.paascloud.net/paascloud/doc/message_3-2.png)
 
1. 下游应用监听 MQ 消息组件并获取消息, 并存储本地消息
2. 下游系统通知可靠消息服务已接收到消息
3. 可靠消息把消息更新为已接收状态
4. 下游应用根据 MQ 消息体信息处理本地业务
5. 下游应用向 MQ 组件自动发送 ACK 确认消息被消费
6. 下游应用通知可靠消息系统消息被成功消费，可靠消息将该消息状态更改为以消费,任务表状态修改为已完成。

失败步骤 | 现象 | 一致性
---|--- |---
第1步 | 下游应用业务未接收MQ消息,MQ消息为已发送未接收 | 不一致
第2步 | 通知可靠消息服务，接收到消息 | 不一致
第3步 | 下游应用异步通知 | 不一致
第4步 | 下游应用数据回滚,本地消息存储成功,消息状态为已接收未成功消费 | 一致
第5步 | MQ未收到ack确认 | 一致
第6步 | 下游应用异步通知 | 不一致

1. 下游应用监听 MQ 消息组件并获取消息, 并存储本地消息
2. 下游系统通知可靠消息服务已接收到消息
3. 可靠消息把消息更新为已接收状态
4. 下游应用根据 MQ 消息体信息处理本地业务
5. 下游应用向 MQ 组件自动发送 ACK 确认消息被消费
6. 下游应用通知可靠消息系统消息被成功消费，可靠消息将该消息状态更改为已消费,任务表状态修改为已完成

## 3.3 生产者消息状态确认
可靠消息服务定时监听消息的状态，如果存在状态为待确认并且超时的消息，则表示上游应用和可靠消息交互中的步骤 4 或者 5 出现异常。

可靠消息则携带消息体内的信息向上游应用发起请求查询该业务是否已执行。上游应用提供一个可查询接口供可靠消息追溯业务执行状态，如果业务执行成功则更改消息状态为已发送，否则删除此消息确保数据一致。具体流程如下:

![这里写图片描述](https://images-cdn.shimo.im/iAZSZoyfdbYn05ns/image_5.png!thumbnail)


## 3.4 消费者消息状态确认
下游消费MQ服务异步通知可靠消息的过程中可能出现异常,在此可能导致两个现象一、消息已接到但可靠消息没有确认接到二、消息已成功消费但可靠消息没有确认接到,为此下游系统需要提供消费者消息状态查询接口,从而可靠消息重新确认.在确认过程中如果是可靠消息为已消费而下游消费系统为已接收则不进行更新操作. 具体流程如下：

![这里写图片描述](http://img.paascloud.net/paascloud/doc/message-3.4.png)

## 3.5 消息重投
消息已发送则表示上游应用已经执行，接下来则确保下游应用也能正常执行。
可靠消息服务发现可靠消息服务中存在消息状态为已发送并且超时的消息，则表示可靠消息服务和下游应用中存在异常的步骤，无论哪个步骤出现异常，可靠消息服务都将此消息重新投递到 MQ 组件中供下游应用监听。
下游应用监听到此消息后，在保证幂等性的情况下重新执行业务并通知可靠消息服务此消息已经成功消费，最终确保上游应用、下游应用的数据最终一致性。具体流程如下：

![这里写图片描述](http://img.paascloud.net/paascloud/doc/message-3.5.png)

1. 可靠消息服务定时查询状态为已发送并超时的消息
2. 可靠消息将消息重新投递到 MQ 组件中
3. 下游应用监听消息，在满足幂等性的条件下，重新执行业务。
4. 下游应用通知可靠消息服务该消息已经成功消费。
5. 更新consumer消息记录为已消费

## 3.6 删除上游系统7天前成功发送的消息
在预发送执行MQ消息的时候本地消息如果落库则需要删除消息,否则业务系统需要额外提供查询消息发送状态接口, 这里介绍两种方法

第一种,RPC服务接口来实现, 在生产者和消费者注册到可靠消息的时候把生产者和消费者存储到BeanFactory的Map里在定时清理任务的时候去处理在线的RPC服务

第二种,发可靠消息来实现, 确保100%到达
## 3.7 删除下游系统7天前成功消费的消息
在消费MQ消息的时候本地消息如果落库则需要删除消息,否则业务系统需要额外提供查询消息发送状态接口,删除实现同3.6
## 3.8 每天备份可靠消息记录
每天将成功消息删除并备份到对应数据库提供历史消息查询功能,当然如果你选择mongo可以不考虑备份消息
# 4 核心代码实现
## 服务注册

```java
public static void startup(PaascloudProperties paascloudProperties, String host, String app) {
	CoordinatorRegistryCenter coordinatorRegistryCenter = createCoordinatorRegistryCenter(paascloudProperties.getZk());
	RegisterDto dto = new RegisterDto(app, host, coordinatorRegistryCenter);
	Long serviceId = new IncrementIdGenerator(dto).nextId();
	IncrementIdGenerator.setServiceId(serviceId);
	registerMq(paascloudProperties, host, app);
}

private static void registerMq(PaascloudProperties paascloudProperties, String host, String app) {
	CoordinatorRegistryCenter coordinatorRegistryCenter = createCoordinatorRegistryCenter(paascloudProperties.getZk());
	AliyunProperties.RocketMqProperties rocketMq = paascloudProperties.getAliyun().getRocketMq();
	String consumerGroup = rocketMq.isReliableMessageConsumer() ? rocketMq.getConsumerGroup() : null;
	String namesrvAddr = rocketMq.getNamesrvAddr();
	String producerGroup = rocketMq.isReliableMessageProducer() ? rocketMq.getProducerGroup() : null;
	coordinatorRegistryCenter.registerMq(app, host, producerGroup, consumerGroup, namesrvAddr);
}
@Override
public void registerMq(final String app, final String host, final String producerGroup, final String consumerGroup, String namesrvAddr) {
	// 注册生产者
	final String producerRootPath = GlobalConstant.ZK_REGISTRY_PRODUCER_ROOT_PATH + GlobalConstant.Symbol.SLASH + app;
	final String consumerRootPath = GlobalConstant.ZK_REGISTRY_CONSUMER_ROOT_PATH + GlobalConstant.Symbol.SLASH + app;
	ReliableMessageRegisterDto dto;
	if (StringUtils.isNotEmpty(producerGroup)) {
		dto = new ReliableMessageRegisterDto().setProducerGroup(producerGroup).setNamesrvAddr(namesrvAddr);
		String producerJson = JSON.toJSONString(dto);
		this.persist(producerRootPath, producerJson);
		this.persistEphemeral(producerRootPath + GlobalConstant.Symbol.SLASH + host, DateUtil.now());
	}
	// 注册消费者
	if (StringUtils.isNotEmpty(consumerGroup)) {
		dto = new ReliableMessageRegisterDto().setConsumerGroup(consumerGroup).setNamesrvAddr(namesrvAddr);
		String producerJson = JSON.toJSONString(dto);
		this.persist(consumerRootPath, producerJson);
		this.persistEphemeral(consumerRootPath + GlobalConstant.Symbol.SLASH + host, DateUtil.now());
	}

}
```

### 消费注解 @MqProducerStore

```java
@Around(value = "mqProducerStoreAnnotationPointcut()")
public Object processMqProducerStoreJoinPoint(ProceedingJoinPoint joinPoint) throws Throwable {
	log.info("processMqProducerStoreJoinPoint - 线程id={}", Thread.currentThread().getId());
	Object result;
	Object[] args = joinPoint.getArgs();
	MqProducerStore annotation = getAnnotation(joinPoint);
	MqSendTypeEnum type = annotation.sendType();
	int orderType = annotation.orderType().orderType();
	DelayLevelEnum delayLevelEnum = annotation.delayLevel();
	if (args.length == 0) {
		throw new TpcBizException(ErrorCodeEnum.TPC10050005);
	}
	MqMessageData domain = null;
	for (Object object : args) {
		if (object instanceof MqMessageData) {
			domain = (MqMessageData) object;
			break;
		}
	}

	if (domain == null) {
		throw new TpcBizException(ErrorCodeEnum.TPC10050005);
	}

	domain.setOrderType(orderType);
	domain.setProducerGroup(producerGroup);
	if (type == MqSendTypeEnum.WAIT_CONFIRM) {
		if (delayLevelEnum != DelayLevelEnum.ZERO) {
			domain.setDelayLevel(delayLevelEnum.delayLevel());
		}
		mqMessageService.saveWaitConfirmMessage(domain);
	}
	result = joinPoint.proceed();
	if (type == MqSendTypeEnum.SAVE_AND_SEND) {
		mqMessageService.saveAndSendMessage(domain);
	} else if (type == MqSendTypeEnum.DIRECT_SEND) {
		mqMessageService.directSendMessage(domain);
	} else {
		mqMessageService.confirmAndSendMessage(domain.getMessageKey());
	}
	return result;
}
```

### 生产注解@MqConsumerStore

```java
@Around(value = "mqConsumerStoreAnnotationPointcut()")
public Object processMqConsumerStoreJoinPoint(ProceedingJoinPoint joinPoint) throws Throwable {

	log.info("processMqConsumerStoreJoinPoint - 线程id={}", Thread.currentThread().getId());
	Object result;
	long startTime = System.currentTimeMillis();
	Object[] args = joinPoint.getArgs();
	MqConsumerStore annotation = getAnnotation(joinPoint);
	boolean isStorePreStatus = annotation.storePreStatus();
	List<MessageExt> messageExtList;
	if (args == null || args.length == 0) {
		throw new TpcBizException(ErrorCodeEnum.TPC10050005);
	}

	if (!(args[0] instanceof List)) {
		throw new TpcBizException(ErrorCodeEnum.GL99990001);
	}

	try {
		messageExtList = (List<MessageExt>) args[0];
	} catch (Exception e) {
		log.error("processMqConsumerStoreJoinPoint={}", e.getMessage(), e);
		throw new TpcBizException(ErrorCodeEnum.GL99990001);
	}

	MqMessageData dto = this.getTpcMqMessageDto(messageExtList.get(0));
	final String messageKey = dto.getMessageKey();
	if (isStorePreStatus) {
		mqMessageService.confirmReceiveMessage(consumerGroup, dto);
	}
	String methodName = joinPoint.getSignature().getName();
	try {
		result = joinPoint.proceed();
		log.info("result={}", result);
		if (CONSUME_SUCCESS.equals(result.toString())) {
			mqMessageService.saveAndConfirmFinishMessage(consumerGroup, messageKey);
		}
	} catch (Exception e) {
		log.error("发送可靠消息, 目标方法[{}], 出现异常={}", methodName, e.getMessage(), e);
		throw e;
	} finally {
		log.info("发送可靠消息 目标方法[{}], 总耗时={}", methodName, System.currentTimeMillis() - startTime);
	}
	return result;
}
```

### 定时清理所有订阅者消费成功的消息数据

```java


@Slf4j
@ElasticJobConfig(cron = "0 0 0 1/1 * ?")
public class DeleteRpcConsumerMessageJob implements SimpleJob {
	@Resource
	private PaascloudProperties paascloudProperties;
	@Resource
	private TpcMqMessageService tpcMqMessageService;

	/**
	 * Execute.
	 *
	 * @param shardingContext the sharding context
	 */
	@Override
	public void execute(final ShardingContext shardingContext) {
		ShardingContextDto shardingContextDto = new ShardingContextDto(shardingContext.getShardingTotalCount(), shardingContext.getShardingItem());
		final TpcMqMessageDto message = new TpcMqMessageDto();
		message.setMessageBody(JSON.toJSONString(shardingContextDto));
		message.setMessageTag(AliyunMqTopicConstants.MqTagEnum.DELETE_CONSUMER_MESSAGE.getTag());
		message.setMessageTopic(AliyunMqTopicConstants.MqTopicEnum.TPC_TOPIC.getTopic());
		message.setProducerGroup(paascloudProperties.getAliyun().getRocketMq().getProducerGroup());
		String refNo = Long.toString(UniqueIdGenerator.generateId());
		message.setRefNo(refNo);
		message.setMessageKey(refNo);
		tpcMqMessageService.saveAndSendMessage(message);
	}
}
```
## 定时清理所有生产者发送成功的消息数据

```java
@Slf4j
@ElasticJobConfig(cron = "0 0 0 1/1 * ?")
public class DeleteRpcExpireFileJob implements SimpleJob {

	@Resource
	private OpcRpcService opcRpcService;

	/**
	 * Execute.
	 *
	 * @param shardingContext the sharding context
	 */
	@Override
	public void execute(final ShardingContext shardingContext) {
		opcRpcService.deleteExpireFile();
	}
}
```
## 定时清理所有生产者发送成功的消息数据

```java

@Slf4j
@ElasticJobConfig(cron = "0 0 1 1/1 * ?")
public class DeleteRpcProducerMessageJob implements SimpleJob {

	@Resource
	private PaascloudProperties paascloudProperties;
	@Resource
	private TpcMqMessageService tpcMqMessageService;

	/**
	 * Execute.
	 *
	 * @param shardingContext the sharding context
	 */
	@Override
	public void execute(final ShardingContext shardingContext) {

		final TpcMqMessageDto message = new TpcMqMessageDto();
		message.setMessageBody(JSON.toJSONString(shardingContext));
		message.setMessageTag(AliyunMqTopicConstants.MqTagEnum.DELETE_PRODUCER_MESSAGE.getTag());
		message.setMessageTopic(AliyunMqTopicConstants.MqTopicEnum.TPC_TOPIC.getTopic());
		message.setProducerGroup(paascloudProperties.getAliyun().getRocketMq().getProducerGroup());
		String refNo = Long.toString(UniqueIdGenerator.generateId());
		message.setRefNo(refNo);
		message.setMessageKey(refNo);
		tpcMqMessageService.saveAndSendMessage(message);
	}
}
```
## 处理发送中的消息数据

```java

@Component
@Slf4j
@ElasticJobConfig(cron = "0/30 * * * * ?", jobParameter = "fetchNum=200")
public class HandleSendingMessageJob extends AbstractBaseDataflowJob<TpcMqMessage> {
	@Resource
	private TpcMqMessageService tpcMqMessageService;
	@Value("${paascloud.message.handleTimeout}")
	private int timeOutMinute;
	@Value("${paascloud.message.maxSendTimes}")
	private int messageMaxSendTimes;

	@Value("${paascloud.message.resendMultiplier}")
	private int messageResendMultiplier;
	@Resource
	private TpcMqConfirmMapper tpcMqConfirmMapper;

	/**
	 * Fetch job data list.
	 *
	 * @param jobParameter the job parameter
	 *
	 * @return the list
	 */
	@Override
	protected List<TpcMqMessage> fetchJobData(JobParameter jobParameter) {
		MessageTaskQueryDto query = new MessageTaskQueryDto();
		query.setCreateTimeBefore(DateUtil.getBeforeTime(timeOutMinute));
		query.setMessageStatus(MqSendStatusEnum.SENDING.sendStatus());
		query.setFetchNum(jobParameter.getFetchNum());
		query.setShardingItem(jobParameter.getShardingItem());
		query.setShardingTotalCount(jobParameter.getShardingTotalCount());
		query.setTaskStatus(JobTaskStatusEnum.TASK_CREATE.status());
		return tpcMqMessageService.listMessageForWaitingProcess(query);
	}

	/**
	 * Process job data.
	 *
	 * @param taskList the task list
	 */
	@Override
	@Transactional(rollbackFor = Exception.class)
	protected void processJobData(List<TpcMqMessage> taskList) {
		for (TpcMqMessage message : taskList) {

			Integer resendTimes = message.getResendTimes();
			if (resendTimes >= messageMaxSendTimes) {
				tpcMqMessageService.setMessageToAlreadyDead(message.getId());
				continue;
			}

			int times = (resendTimes == 0 ? 1 : resendTimes) * messageResendMultiplier;
			long currentTimeInMillis = Calendar.getInstance().getTimeInMillis();
			long needTime = currentTimeInMillis - times * 60 * 1000;
			long hasTime = message.getUpdateTime().getTime();
			// 判断是否达到了可以再次发送的时间条件
			if (hasTime > needTime) {
				log.debug("currentTime[" + com.xiaoleilu.hutool.date.DateUtil.formatDateTime(new Date()) + "],[SENDING]消息上次发送时间[" + com.xiaoleilu.hutool.date.DateUtil.formatDateTime(message.getUpdateTime()) + "],必须过了[" + times + "]分钟才可以再发送。");
				continue;
			}

			// 前置状态
			List<Integer> preStatusList = Lists.newArrayList(JobTaskStatusEnum.TASK_CREATE.status());
			// 设置任务状态为执行中
			message.setPreStatusList(preStatusList);
			message.setTaskStatus(JobTaskStatusEnum.TASK_EXETING.status());
			int updateRes = tpcMqMessageService.updateMqMessageTaskStatus(message);
			if (updateRes > 0) {
				try {

					// 查询是否全部订阅者都确认了消息 是 则更新消息状态完成, 否则重发消息

					int count = tpcMqConfirmMapper.selectUnConsumedCount(message.getMessageKey());
					int status = JobTaskStatusEnum.TASK_CREATE.status();
					if (count < 1) {
						TpcMqMessage update = new TpcMqMessage();
						update.setMessageStatus(MqSendStatusEnum.FINISH.sendStatus());
						update.setId(message.getId());
						tpcMqMessageService.updateMqMessageStatus(update);
						status = JobTaskStatusEnum.TASK_SUCCESS.status();
					} else {
						tpcMqMessageService.resendMessageByMessageId(message.getId());
					}

					// 前置状态
					preStatusList = Lists.newArrayList(JobTaskStatusEnum.TASK_EXETING.status());
					// 设置任务状态为执行中
					message.setPreStatusList(preStatusList);
					message.setTaskStatus(status);
					tpcMqMessageService.updateMqMessageTaskStatus(message);
				} catch (Exception e) {
					log.error("重发失败 ex={}", e.getMessage(), e);
					// 设置任务状态为执行中
					preStatusList = Lists.newArrayList(JobTaskStatusEnum.TASK_EXETING.status());
					message.setPreStatusList(preStatusList);
					message.setTaskStatus(JobTaskStatusEnum.TASK_SUCCESS.status());
					tpcMqMessageService.updateMqMessageTaskStatus(message);
				}
			}
		}
	}
}
```
## 处理待确认的消息数据

```java

@Slf4j
@Component
@ElasticJobConfig(cron = "0 0/10 * * * ?", jobParameter = "fetchNum=1000")
public class HandleWaitingConfirmMessageJob extends AbstractBaseDataflowJob<String> {
	@Resource
	private TpcMqMessageService tpcMqMessageService;
	@Resource
	private UacRpcService uacRpcService;
	@Value("${paascloud.message.handleTimeout}")
	private int timeOutMinute;
	private static final String PID_UAC = "PID_UAC";

	/**
	 * Fetch job data list.
	 *
	 * @param jobParameter the job parameter
	 *
	 * @return the list
	 */
	@Override
	protected List<String> fetchJobData(JobParameter jobParameter) {
		MessageTaskQueryDto query = new MessageTaskQueryDto();
		query.setCreateTimeBefore(DateUtil.getBeforeTime(timeOutMinute));
		query.setMessageStatus(MqSendStatusEnum.WAIT_SEND.sendStatus());
		query.setFetchNum(jobParameter.getFetchNum());
		query.setShardingItem(jobParameter.getShardingItem());
		query.setShardingTotalCount(jobParameter.getShardingTotalCount());
		query.setTaskStatus(JobTaskStatusEnum.TASK_CREATE.status());
		query.setProducerGroup(PID_UAC);
		return tpcMqMessageService.queryWaitingConfirmMessageKeyList(query);
	}

	/**
	 *
	 */
	@Override
	protected void processJobData(List<String> messageKeyList) {
		if (messageKeyList == null) {
			return;
		}
		List<String> resendMessageList = uacRpcService.queryWaitingConfirmMessageKeyList(messageKeyList);
		if (resendMessageList == null) {
			resendMessageList = Lists.newArrayList();
		}
		messageKeyList.removeAll(resendMessageList);
		tpcMqMessageService.handleWaitingConfirmMessage(messageKeyList, resendMessageList);
	}
}
```

## 可靠消息用法
例子
```java

@MqProducerStore
	public void resetLoginPwd(final MqMessageData mqMessageData, final UacUser update) {
		log.info("重置密码. mqMessageData={}, user={}", mqMessageData, update);
		int updateResult = uacUserMapper.updateByPrimaryKeySelective(update);
		if (updateResult < 1) {
			log.error("用户【 {} 】重置密码失败", update.getLoginName());
		} else {
			log.info("用户【 {} 】重置密码失败", update.getLoginName());
		}
	}
```
强制： 需要使用的使用加上述两个注解，方法参数需要加入 MqMessageData 