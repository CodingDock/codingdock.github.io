# RocketMQ自动创建Topic的QueueNum为什么总是4

## 问题描述

- 如题

## 版本

- 4.4.0

## 自动Topic创建过程跟踪

- Broker接收到发送者发送的请求后会路由到`SendMessageProcessor.processRequest()`方法处理。

- 然后会进入`SendMessageProcessor.sendMessage()`方法。

- 然后在进入其父类的`AbstractSendMessageProcessor.msgCheck()`进行校验。

- 再进入`TopicConfigManager.createTopicInSendMessageMethod()`方法，此时方法入参为

  ```java
  final String topic:待创建的Topic
  final String defaultTopic:默认Topic，此处为：TBW102
  final String remoteAddress:producer的地址
  final int clientDefaultTopicQueueNums:默认Topic的QueueNum。这个很关键，此处为4.
  final int topicSysFlag:是否包含RETRY_GROUP_TOPIC_PREFIX字符，此处为0
  ```

- 方法内部关于TopicQueueNum的逻辑如下：

  ```java
  int queueNums = 
      clientDefaultTopicQueueNums > defaultTopicConfig.getWriteQueueNums() 
      ? defaultTopicConfig.getWriteQueueNums() :  clientDefaultTopicQueueNums;
  
  if (queueNums < 0) {
      queueNums = 0;
  }
  
  topicConfig.setReadQueueNums(queueNums);
  topicConfig.setWriteQueueNums(queueNums);
  ```

  解释：此处，clientDefaultTopicQueueNums=4，defaultTopicConfig是`TBW102`的配置。在RocketMQ的存储目录`store/config/topics.json`可以看到具体配置，其TopicQueueNum=8。这个`TBW102`Topc是RocketMQ自动生成的，用来当模板用。所以这段逻辑运行下来，queueNums=4。

  那么这个4从何而来？进入`msgCheck()`方法看下，

  ```java
  topicConfig = this.brokerController.getTopicConfigManager().createTopicInSendMessageMethod(
                  requestHeader.getTopic(),
                  requestHeader.getDefaultTopic(),
                  RemotingHelper.parseChannelRemoteAddr(ctx.channel()),
                  requestHeader.getDefaultTopicQueueNums(), topicSysFlag);
  ```

  由代码`requestHeader.getDefaultTopicQueueNums()`，可知是requestHeader传过来的。

  这个requestHeader是producer端构建的。进入发送代码看看。确定位置在`DefaultMQProducerImpl.sendKernelImpl()`。

  ```java
  requestHeader.setDefaultTopicQueueNums(
                      this.defaultMQProducer.getDefaultTopicQueueNums());
  
  ```

  跟进去，代码如下：

  ```java
  public class DefaultMQProducer extends ClientConfig implements MQProducer {
      .....
      
      /**
       * Number of queues to create per default topic.
       */
      private volatile int defaultTopicQueueNums = 4;
      
      public int getDefaultTopicQueueNums() {
          return defaultTopicQueueNums;
      }
      ....
  ```

这就是自动创建Topic的QueueNum来源。

## 总结

关于Topic的QueueNum有几个地方有默认配置，

- `TopicConfig`里的是16
- 模板Topic `TBW102` 里的是8
- DefaultMQProducer 里的是4

最终使用的是DefaultMQProducer 里的配置。

**线上建议不要开启自动创建Topic。风险大，万一手抖……，而且个人认为这个QueueNum的确定逻辑说实话有点莫名其妙。**