
Kafka Consumers
==========

**Dependency**

```
<properties>
    <java. version>1.8</java. version>
    <dockerplugin.version>0.4.10</dockerplugin.version>
    <docker.repository>------</docker.repository>
    <compiler.version>3.7.0</compiler.version>
    <lombook.version>1.18.22</lombook.version>
    <springbootdep.version>2.6.6</springbootdep.version>
    <redisson.version>3.19.2</redisson. version>
    <snappy.version>1.1.10.1</snappy. version>
    <google.gson>2.9.1</google.gson>
    <org-json>20230227</org.json>
    <springdoc.version>1.6.7</springdoc. version>
    <kafka.version>0.10.2.1</kafka.version>
    <httpclient.version>4.5</httpclient. version>
    <accounting.acc.commons>1.0.60≤/accounting.acc.commons>
    <mysql.connector.version>8.0.28</mysql.connector.version>
    <project.build.sourceEnceding>UTF-8</project.build.sourceEncoding>
</properties>
```
```
<dependency>
    <groupId>org.apache.kafka</groupId> 
    <artifactId>kafka_2.12</artifactId>
    <version>${kafka.version}</version>
</dependency>
<dependency>
    <groupId>org.apache.kafka</groupId> 
    <artifactId>kafka-clients</artifactId> 
    <version>${kafka.version}</version>
</dependency>
```

**Batch Consumer**

```
import com.fasterxml.jackson.databind.ObjectMapper;
import lombok.extern.slf4j.Slf4j;
import org.apache.kafka.clients.consumer.ConsumerConfig;
import org.apache.kafka.clients.consumer.ConsumerRecord;
import org.apache.kafka.clients.consumer.ConsumerRecords;
import org.apache.kafka.clients.consumer.KafkaConsumer;
import org.apache.kafka.common.serialization.ByteArrayDeserializer;
import org.apache.kafka.common.serialization.StringDeserializer;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.stereotype.Component;

import java.util.Collections;
import java.util.HashMap;
import java.util.Map;
import java.util.concurrent.ExecutorService;

@Slf4j
@Component
public class StatusConsumer extends Thread {

    @Value("${status.consumer.topic}")
    private String statusConsumerTopic;

    @Value("${status.max.poll.interval}")
    private int statusMaxPollInterval;

    @Value("${status.max.poll.records}")
    private int statusMaxPollRecords;

    @Value("${status.bootstrap.servers}")
    private String statusBootstrapServers;

    @Value("${status.group.id}")
    private String statusGroupId;

    @Value("${status.event.auto.commit.interval}")
    private int autoCommitInterval;

    private KafkaConsumer<byte[], String> consumer;

    private boolean interrupted;
    private boolean initialized = false;

    private static ObjectMapper mapper = new ObjectMapper();

    @Autowired
    private ExecutorService statusConsumerExecutorService;

    @Autowired
    private StatusEventProcessor statusEventProcessor;

    public void init() {
        if (initialized) {
            return;
        }
        try {
            interrupted = false;
            log.info("trying to initialize status Consumer . ");
            consumer = new KafkaConsumer<>(getPropertiesMap());
            consumer.subscribe(Collections.singletonList(statusConsumerTopic));
            log.info("status consumer initailized ");
            initialized = true;
        } catch(Exception exception) {
            log.error("Exception in Initializing Kafka Client ", exception);
        }
    }

    private Map<String, Object> getPropertiesMap() {
        Map<String, Object> props = new HashMap<>();
        props.put(org.apache.kafka.clients.consumer.ConsumerConfig.BOOTSTRAP_SERVERS_CONFIG, statusBootstrapServers);
        props.put(org.apache.kafka.clients.consumer.ConsumerConfig.GROUP_ID_CONFIG, statusGroupId);
        props.put(org.apache.kafka.clients.consumer.ConsumerConfig.ENABLE_AUTO_COMMIT_CONFIG, true);
        props.put(org.apache.kafka.clients.consumer.ConsumerConfig.AUTO_COMMIT_INTERVAL_MS_CONFIG, autoCommitInterval);
        props.put(ConsumerConfig.MAX_POLL_INTERVAL_MS_CONFIG, statusMaxPollInterval);
        props.put(org.apache.kafka.clients.consumer.ConsumerConfig.MAX_POLL_RECORDS_CONFIG, statusMaxPollRecords);
        props.put(org.apache.kafka.clients.consumer.ConsumerConfig.KEY_DESERIALIZER_CLASS_CONFIG, ByteArrayDeserializer.class);
        props.put(org.apache.kafka.clients.consumer.ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG, StringDeserializer.class);
        return props;
    }

    public void close() {
        consumer.close();
    }


    public boolean isRunning() {
        return consumer != null;
    }

    public void shutdown() {
        if (consumer != null) {
            log.info("Shutting down Kafka Server");
            consumer.close();
            consumer = null;
            initialized = false;
        }
    }

    @Override
    public void run() {

        while(!Thread.currentThread().isInterrupted() && !interrupted) {
            try {
                ConsumerRecords<byte[], String> consumerRecords = consumer.poll(100);
                log.info("Records polled: " + consumerRecords.count());

                for (ConsumerRecord<byte[], String> record : consumerRecords) {
                    try {
                        log.info("Status event consumer. received event "+ record.value());
                        OnlinePaymentUpdateRequest statusData = mapper.readValue(record.value(), OnlinePaymentUpdateRequest.class);
                        if (statusData != null) {
                            if (statusConsumerExecutorService != null) {
                                statusConsumerExecutorService.submit(() -> statusEventProcessor.process(statusData));
                            } else {
                                statusEventProcessor.process(statusData);
                            }
                        }
                    } catch(Exception ex) {
                        log.error("Exception in processing kafka Event ", ex);
                    }
                }
            } catch (Exception exception) {
                if(exception.getCause() instanceof InterruptedException) {
                    interrupted = true;
                    break;
                } else {
                    shutdown();
                    init();
                }
            }
        }
        shutdown();
    }
}

```

============


```
import lombok.extern.slf4j.Slf4j;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.stereotype.Component;

import javax.annotation.PostConstruct;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Future;

@Slf4j
@Component
public class StatusConsumerInitializer {

    @Value("${status.consumer.start}")
    private boolean statusConsumerStart;

    @Autowired
    private StatusConsumer statusConsumer;

    @Autowired
    private ExecutorService consumerExecutorService;

    private Future statusConsumerFuture = null;

    @PostConstruct
    public void init(){
        try{
            if(statusConsumerStart){
                statusConsumer.init();
                statusConsumerFuture = consumerExecutorService.submit(statusConsumer);

                if (!statusConsumer.isRunning() || statusConsumerFuture == null){
                    log.error("Exception in Initializing Kafka Client ");
                    throw new RuntimeException("error initializing status event consumer");
                }
                log.info("started status consumers");

            }
        } catch (Exception e){
            log.error("Exception in Starting Kafka consumer ", e);
        }
    }
}
```


=============

```
import org.springframework.beans.factory.annotation.Value;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

import java.util.concurrent.ExecutorService;
import java.util.concurrent.LinkedBlockingDeque;
import java.util.concurrent.ThreadPoolExecutor;
import java.util.concurrent.TimeUnit;

@Configuration
public class ThreadPoolExecutorConfig {

    @Value("${executor.core.pool.size}")
    private int poolSize;

    @Value("${maximum.pool.size}")
    private int maximumPoolSize;

    @Value("${keep.alive.time.sec}")
    private int keepAliveTimeSec;

    @Bean("executorService")
    public ExecutorService getExecutorService(){
        ExecutorService executorService = new ThreadPoolExecutor(poolSize, maximumPoolSize, keepAliveTimeSec,
                TimeUnit.SECONDS, new LinkedBlockingDeque<>());
        return executorService;
    }
}

```


=============


```
import lombok.extern.slf4j.Slf4j;
import org.apache.commons.lang3.StringUtils;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.stereotype.Component;

@Slf4j
@Component
public class StatusEventProcessor {

    @Autowired
    OnlinePaymentService onlinePaymentService;

    @Autowired
    ProducerPool producerPool;

    @Value("${status.consumer.topic}")
    private String statusConsumerTopic;

    public void process(OnlinePaymentUpdateRequest updateRequest) {
        try {
            String status = onlinePaymentService.updatePayment(updateRequest.getMerchantPaymentRef(), updateRequest.getMerchant(), updateRequest);
            /*if (!StringUtils.equalsIgnoreCase("SUCCESS", status)) {
                producerPool.send(statusConsumerTopic, updateRequest);
            }*/
        } catch (Exception e) {
            log.error("Exception in processing kafka Event ", e);
        }
    }
}
```


=============


```
import com.fasterxml.jackson.databind.ObjectMapper;
import lombok.extern.slf4j.Slf4j;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Component;

@Slf4j
@Component
public class ProducerPool implements IProducerPool {

    @Autowired
    private ProducerClient producerClient;

    private static ObjectMapper mapper = new ObjectMapper();

    @Override
    public void send(String topic, OnlinePaymentUpdateRequest data) throws Exception {
        try {
            String value = mapper.writeValueAsString(data);
            producerClient.send(topic, value);
            log.info("SUCCESS IN PUSHING EVENT " + value);
        } catch (Exception exception) {
            log.error("Exception in deserializing Kafka message ", exception);
            throw new Exception(exception);
        }
    }
}
```


=============


```
import lombok.extern.slf4j.Slf4j;
import org.apache.kafka.clients.producer.KafkaProducer;
import org.apache.kafka.clients.producer.Producer;
import org.apache.kafka.clients.producer.ProducerConfig;
import org.apache.kafka.clients.producer.ProducerRecord;
import org.apache.kafka.common.serialization.StringSerializer;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.stereotype.Component;

import javax.annotation.PostConstruct;
import java.util.Properties;

@Slf4j
@Component
public class ProducerClient implements IProducerClient {


    @Value("${producer.broker.list}")
    String brokerList;

    @Value("${producer.request.ack}")
    String requestAck;

    @Value("${producer.request.timeout.ms}")
    String requestTimeout;

    Producer<String, String> producer;

    @PostConstruct
    public void init() {
        Properties properties = new Properties();
        properties.put(ProducerConfig.BOOTSTRAP_SERVERS_CONFIG, brokerList);
        properties.put(ProducerConfig.ACKS_CONFIG, requestAck);
        producer = new KafkaProducer<String, String>(properties, new StringSerializer(), new StringSerializer());
    }


    public void shutdown() {
        producer.close();
    }

    @Override
    public void send(String topic, String value) throws Exception {
        try {
            ProducerRecord<String, String> record = new ProducerRecord<>(topic, value);
            producer.send(record);
        } catch (Exception exception) {
            log.error("Exception while producing data ", exception);
            throw new Exception(exception);
        }
    }


    @Override
    public void send(String topic, String key, String value) {
        try {
            ProducerRecord<String, String> record = new ProducerRecord<>(topic, key, value);
            producer.send(record);
        } catch (Exception exception) {
            log.error("Exception while producing data ", exception);
        }
    }
}
```


==========


```
public interface IProducerClient {
    public void send(String topic, String value) throws Exception;
    public void send(String topic, String key, String value);
}
```


==========

```
public interface IProducerPool {
    public void send(String topic, OnlinePaymentUpdateRequest data) throws Exception;
}
```


==========


```
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.stereotype.Component;

@Component
public class KafkaProduceHandler {


    @Value("${kafka.producer.push.enable}")
    private boolean kafkaProducerPushEnable;

    @Autowired
    private ProducerPool producerPool;

    public void pushToKafka(OnlinePaymentUpdateRequest data, String topic) throws Exception {
        if (kafkaProducerPushEnable) {
            producerPool.send(topic,data);
        }


    }
}
```

===========

```
kafka.producer.push.enable=true
producer.broker.list=localhost\:9092
producer.request.ack=1
producer.request.timeout.ms=100
status.consumer.start=true
status.consumer.topic=hyper_charger_billing_status
status.max.poll.interval=60000
status.max.poll.records=100
status.bootstrap.servers=localhost\:9092
status.group.id=status_consumer_group
status.event.auto.commit.interval=100
```


=========================================================


**Single event consumer**

```
import com.fasterxml.jackson.databind.ObjectMapper;
import kafka.consumer.ConsumerConfig;
import kafka.consumer.ConsumerIterator;
import kafka.consumer.KafkaStream;
import kafka.javaapi.consumer.ConsumerConnector;
import kafka.message.MessageAndMetadata;
import org.apache.commons.lang3.StringUtils;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.stereotype.Component;

import java.util.*;
import java.util.concurrent.Callable;

@Component("deviceLocationConsumer")
public class DeviceLocationConsumer implements Callable {

    @Value("${device.location.zookeeper.connect}")
    private String zookeeperConnect;

    @Value("${device.location.auto.offset.reset}")
    private String autoOffsetReset;

    @Value("${device.location.auto.commit.enable}")
    private String autoCommitEnable;

    @Value("${device.location.topic}")
    private String deviceLocationTopic;

    @Value("${device.location.topic.consumer.count}")
    private int deviceLocationTopicConsumerCount;

    @Value("${device.location.group.id}")
    private String deviceLocationGroupId;

    @Value("${ev.bike.nudge.supported.categories}")
    private String evNudgeSupportedCategories;

    private ConsumerConnector consumer;
    private ConsumerIterator<byte[], byte[]> consumerIterator;
    private boolean interrupted;
    private boolean initialized = false;

    private Logger logger = LoggerFactory.getLogger(DeviceLocationConsumer.class);
    private ObjectMapper mapper = new ObjectMapper();

    @Autowired
    private DeviceLocationProcessor deviceLocationProcessor;

    @Autowired
    private EVBikeEventProcessor evBikeEventProcessor;

    public void init() {
        if (initialized) {
            return;
        }
        try {
            interrupted = false;
            consumer =  kafka.consumer.Consumer.createJavaConsumerConnector(createConsumerConfig());
            logger.info("device location event kafka consumer initialized.");
            Map<String, Integer> topicCountMap = new HashMap<String, Integer>();
            topicCountMap.put(deviceLocationTopic, deviceLocationTopicConsumerCount);
            Map<String, List<KafkaStream<byte[], byte[]>>> consumerMap = consumer.createMessageStreams(topicCountMap);
            KafkaStream<byte[], byte[]> stream = consumerMap.get(deviceLocationTopic).get(0);
            consumerIterator = stream.iterator();
            initialized = true;
        } catch(Exception exception) {
            logger.error("initialization failed for device location event kafka consumer", exception);
        }
    }

    private ConsumerConfig createConsumerConfig() {
        Properties props = new Properties();
        props.put("zookeeper.connect", zookeeperConnect);
        props.put("group.id", deviceLocationGroupId);
        props.put("auto.offset.reset", autoOffsetReset);
        props.put("auto.commit.enable", autoCommitEnable);
        return new ConsumerConfig(props);
    }

    public void shutdown() {
        if (consumer != null) {
            logger.error("shutting down device location event kafka consumer");
            consumer.shutdown();
            consumer = null;
            initialized = false;
        }
    }

    public boolean isRunning() {
        return consumer != null;
    }

    public Object call() throws Exception {
        while(!Thread.currentThread().isInterrupted() && !interrupted) {
            try {
                if (consumerIterator.hasNext()) {

                    MessageAndMetadata<byte[], byte[]> messageAndMetaData = consumerIterator.next();
                    byte[] eventBytes = messageAndMetaData.message();
                    if (eventBytes == null){
                        logger.error("no event received on topic - {}", deviceLocationTopic);
                        continue;
                    }

                    String event = new String(eventBytes).trim();
                    logger.info("device location consumer. received event - {}", event);
                    try {
                        DeviceLocationPing deviceLocationPing = mapper.readValue(event, DeviceLocationPing.class);
                        DeviceLocationPing.DeviceLocationPingData deviceLocationPingData = deviceLocationPing.getDeviceLocationPingData();
                        DevicePingData devicePingData = mapper.readValue(deviceLocationPingData.getLocationJson(), DevicePingData.class);
                        deviceLocationProcessor.process(devicePingData);
                    }
                    catch (Exception exception) {
                        logger.error("error in processing event - {} , error - {}", event, exception);
                    }
                    consumer.commitOffsets();
                }
                else {
                    logger.info("topic - {}, nothing to read from consumer.", deviceLocationTopic);
                }
            }
            catch (Exception exception) {
                logger.error("topic - {}, exception while reading records from kafka queue - {}", deviceLocationTopic, exception);
                if(exception.getCause() instanceof InterruptedException) {
                    logger.error("topic - {}, consumer interrupted. breaking it.", deviceLocationTopic);
                    interrupted = true;
                    break;
                }
                else {
                    logger.error("topic - {}, restarting kafka consumer because of error : {}", deviceLocationTopic, exception.getMessage());
                    shutdown();
                    init();
                }
            }
        }
        logger.error("topic - {}, transaction events consumer interrupted.", deviceLocationTopic);
        shutdown();
        return true;
    }

}
```
