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
