---
layout:     post 
title:      Apache Flume源码学习 
subtitle:   Kafka Source技术解析 
date:       2021-03-12 
author:     假正经
header-img: img/avatar-ny.jpg 
catalog: true 
tags:
    - Apache Flume 
    - Apache Kafka
---

本篇主要摘取部分Flume源码进行分析学习,代码略微多一些,建议耐心查看

# Kafka Source
> package: org.apache.flume.source.kafka.KafkaSource

##### 变量

```
// Constants used only for offset migration zookeeper connections
  private static final int ZK_SESSION_TIMEOUT = 30000;
  private static final int ZK_CONNECTION_TIMEOUT = 30000;

  private Context context;
  private Properties kafkaProps;
  private KafkaSourceCounter counter;
  private KafkaConsumer<String, byte[]> consumer;
  private Iterator<ConsumerRecord<String, byte[]>> it;

  private final List<Event> eventList = new ArrayList<Event>();
  private Map<TopicPartition, OffsetAndMetadata> tpAndOffsetMetadata;
  private AtomicBoolean rebalanceFlag;

  private Map<String, String> headers;

  private Optional<SpecificDatumReader<AvroFlumeEvent>> reader = Optional.absent();
  private BinaryDecoder decoder = null;

  private boolean useAvroEventFormat;

  private int batchUpperLimit;
  private int maxBatchDurationMillis; //最大拉取数据延迟等待时间

  private Subscriber subscriber;

  private String zookeeperConnect;
  private String bootstrapServers;
  private String groupId = DEFAULT_GROUP_ID; //kafka cousume group_id is ‘flume’
  private boolean migrateZookeeperOffsets = DEFAULT_MIGRATE_ZOOKEEPER_OFFSETS;

```

##### Apache Kafka Topic订阅接口

```java
public abstract class Subscriber<T> {
    /**
     * 消费者订阅Topic数据 ，并添加Listener 监听数据订阅balance
     */
    public abstract void subscribe(KafkaConsumer<?, ?> consumer, SourceRebalanceListener listener);

    /**
     获取topics
     */
    public T get() {
        return null;
    }
}
```

##### 两大订阅方式实现

```java
private class TopicListSubscriber extends Subscriber<List<String>> {
    private List<String> topicList;

    public TopicListSubscriber(String commaSeparatedTopics) {
        this.topicList = Arrays.asList(commaSeparatedTopics.split("^\\s+|\\s*,\\s*|\\s+$"));
    }

    @Override
    public void subscribe(KafkaConsumer<?, ?> consumer, SourceRebalanceListener listener) {
        consumer.subscribe(topicList, listener);
    }

    @Override
    public List<String> get() {
        return topicList;
    }
}

private class PatternSubscriber extends Subscriber<Pattern> {
    private Pattern pattern;

    public PatternSubscriber(String regex) {
        this.pattern = Pattern.compile(regex);
    }

    @Override
    public void subscribe(KafkaConsumer<?, ?> consumer, SourceRebalanceListener listener) {
        consumer.subscribe(pattern, listener);
    }

    @Override
    public Pattern get() {
        return pattern;
    }
}
```

##### Topic数据订阅启动入口

```
  @Override
  protected Status doProcess() throws EventDeliveryException {
    final String batchUUID = UUID.randomUUID().toString();
    byte[] kafkaMessage;
    String kafkaKey;
    Event event;
    byte[] eventBody;

    try {
      // prepare time variables for new batch
      final long nanoBatchStartTime = System.nanoTime();
      final long batchStartTime = System.currentTimeMillis();
      final long maxBatchEndTime = System.currentTimeMillis() + maxBatchDurationMillis;

      while (eventList.size() < batchUpperLimit && System.currentTimeMillis() < maxBatchEndTime) {

        if (it == null || !it.hasNext()) {
          // Obtaining new records
          // Poll time is remainder time for current batch. 拉取数据
          ConsumerRecords<String, byte[]> records = consumer.poll(Math.max(0, maxBatchEndTime - System.currentTimeMillis()));
          it = records.iterator();

          // this flag is set to true in a callback when some partitions are revoked.
          // If there are any records we commit them.
          if (rebalanceFlag.get()) {
            rebalanceFlag.set(false);
            break;
          }
          // check records after poll
          if (!it.hasNext()) {
            if (log.isDebugEnabled()) {
              counter.incrementKafkaEmptyCount();
              log.debug("Returning with backoff. No more data to read");
            }
            // batch time exceeded
            break;
          }
        }

        // get message
        ConsumerRecord<String, byte[]> message = it.next();
        kafkaKey = message.key();
        kafkaMessage = message.value();

        //默认DEFAULT_AVRO_EVENT为false
        if (useAvroEventFormat) {
          //Assume the event is in Avro format using the AvroFlumeEvent schema
          //Will need to catch the exception if it is not
          ByteArrayInputStream in = new ByteArrayInputStream(message.value());
          decoder = DecoderFactory.get().directBinaryDecoder(in, decoder);
          if (!reader.isPresent()) {
            reader = Optional.of(new SpecificDatumReader<AvroFlumeEvent>(AvroFlumeEvent.class));
          }
          //This may throw an exception but it will be caught by the
          //exception handler below and logged at error
          AvroFlumeEvent avroevent = reader.get().read(null, decoder);

          eventBody = avroevent.getBody().array();
          headers = toStringMap(avroevent.getHeaders());
        } else {
          eventBody = message.value();
          headers.clear();
          headers = new HashMap<String, String>(4);
        }

        // Add headers to event (timestamp, topic, partition, key) only if they don't exist
        if (!headers.containsKey(KafkaSourceConstants.TIMESTAMP_HEADER)) {
          headers.put(KafkaSourceConstants.TIMESTAMP_HEADER,
              String.valueOf(System.currentTimeMillis()));
        }
        if (!headers.containsKey(KafkaSourceConstants.TOPIC_HEADER)) {
          headers.put(KafkaSourceConstants.TOPIC_HEADER, message.topic());
        }
        if (!headers.containsKey(KafkaSourceConstants.PARTITION_HEADER)) {
          headers.put(KafkaSourceConstants.PARTITION_HEADER,
              String.valueOf(message.partition()));
        }

        if (kafkaKey != null) {
          headers.put(KafkaSourceConstants.KEY_HEADER, kafkaKey);
        }

        if (log.isTraceEnabled()) {
          if (LogPrivacyUtil.allowLogRawData()) {
            log.trace("Topic: {} Partition: {} Message: {}", new String[]{
                message.topic(),
                String.valueOf(message.partition()),
                new String(eventBody)
            });
          } else {
            log.trace("Topic: {} Partition: {} Message arrived.",
                message.topic(),
                String.valueOf(message.partition()));
          }
        }

        event = EventBuilder.withBody(eventBody, headers);
        eventList.add(event);

        if (log.isDebugEnabled()) {
          log.debug("Waited: {} ", System.currentTimeMillis() - batchStartTime);
          log.debug("Event #: {}", eventList.size());
        }

        // For each partition store next offset that is going to be read.
        tpAndOffsetMetadata.put(new TopicPartition(message.topic(), message.partition()),
                new OffsetAndMetadata(message.offset() + 1, batchUUID));
      }

      //将source data add to channel processor
      if (eventList.size() > 0) {
        counter.addToKafkaEventGetTimer((System.nanoTime() - nanoBatchStartTime) / (1000 * 1000));
        counter.addToEventReceivedCount((long) eventList.size());
        getChannelProcessor().processEventBatch(eventList);
        counter.addToEventAcceptedCount(eventList.size());
        if (log.isDebugEnabled()) {
          log.debug("Wrote {} events to channel", eventList.size());
        }
        eventList.clear();

        //同步更新topic offect
        if (!tpAndOffsetMetadata.isEmpty()) {
          long commitStartTime = System.nanoTime();
          consumer.commitSync(tpAndOffsetMetadata);
          long commitEndTime = System.nanoTime();
          counter.addToKafkaCommitTimer((commitEndTime - commitStartTime) / (1000 * 1000));
          tpAndOffsetMetadata.clear();
        }
        return Status.READY;
      }

      return Status.BACKOFF;
    } catch (Exception e) {
      log.error("KafkaSource EXCEPTION, {}", e);
      return Status.BACKOFF;
    }
  }
```

##### PollableSourceRunner创建内部线程类实现数据实时获取

```
package org.apache.flume.source;

@Override
public void start() {
    ...

    runner = new PollingRunner();

    runner.source=source;
    runner.counterGroup=counterGroup;
    runner.shouldStop=shouldStop;

    runnerThread=new Thread(runner);
    runnerThread.setName(getClass().getSimpleName()+"-"+
    source.getClass().getSimpleName()+"-"+source.getName());
    runnerThread.start();

    lifecycleState=LifecycleState.START;
}

@Override
public void stop() {
    runner.shouldStop.set(true);
    ...
}

public static class PollingRunner implements Runnable {
    private PollableSource source;
    private AtomicBoolean shouldStop;
    private CounterGroup counterGroup;

    @Override
    public void run() {
        logger.debug("Polling runner starting. Source:{}", source);

        //若未执行stop()行为,会持续循环下去
        while (!shouldStop.get()) {
            counterGroup.incrementAndGet("runner.polls");
            try {
                //source.process会通过多态特性,经过抽象类AbstractPollableSource,直至KafkaSource.doProcess()
                if (source.process().equals(PollableSource.Status.BACKOFF)) {
                    counterGroup.incrementAndGet("runner.backoffs");
                    Thread.sleep(Math.min(counterGroup.incrementAndGet("runner.backoffs.consecutive")
                                    * source.getBackOffSleepIncrement(), source.getMaxBackOffSleepInterval()));
                } else {
                    counterGroup.set("runner.backoffs.consecutive", 0L);
                }
            }...
        }
        logger.debug("Polling runner exiting. Metrics:{}", counterGroup);
    }
}

```
---

### 总结
从Flume实时采集Source端数据的实现方案上看,目前采用 **<font color="#FF6651">线程</font>** + **<font color="#FF6651">多层while循环</font>** 的方式poll **<font color="#FF6651">Apache Kafka</font>** 的数据
代码整体上并不复杂,实现实时处理操作可以借鉴上述的思路

> [Apache Flume Document](http://flume.apache.org/documentation.html) | [Apache Kafka Doc](http://kafka.apache.org/)