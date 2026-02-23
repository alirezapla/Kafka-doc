# Apache Kafka Integration in the Java Spring Ecosystem


## Technical Reference Document

**Version:** Spring for Apache Kafka 3.3.x / Spring Boot 3.2.x  
**Target Audience:** Backend Engineers, System Architects, DevOps Engineers  
**Last Updated:** February 2025

---

## Table of Contents

1. [Introduction & Architecture Overview](#1-introduction--architecture-overview)
2. [Core Components & Dependencies](#2-core-components--dependencies)
3. [Configuration & Setup](#3-configuration--setup)
4. [Producer Implementation Patterns](#4-producer-implementation-patterns)
5. [Consumer Implementation Patterns](#5-consumer-implementation-patterns)
6. [Kafka Streams Integration](#6-kafka-streams-integration)
7. [Spring Cloud Stream Abstraction](#7-spring-cloud-stream-abstraction)
8. [Transaction Management](#8-transaction-management)
9. [Error Handling & Resilience](#9-error-handling--resilience)
10. [Security Implementation](#10-security-implementation)
11. [Testing Strategies](#11-testing-strategies)
12. [Monitoring & Observability](#12-monitoring--observability)
13. [Performance Tuning](#13-performance-tuning)
14. [Deployment Best Practices](#14-deployment-best-practices)

---

## 1. Introduction & Architecture Overview

### 1.1 Spring Kafka Ecosystem

The Spring ecosystem provides multiple abstraction layers for Apache Kafka integration:

| Layer | Purpose | Use Case |
|-------|---------|----------|
| **Spring for Apache Kafka** | Core integration (KafkaTemplate, @KafkaListener) | Direct Kafka operations, fine-grained control |
| **Spring Cloud Stream** | Binder abstraction (functional programming model) | Event-driven microservices, cloud-native apps |
| **Kafka Streams Binder** | Stream processing abstraction | Stateful stream processing, windowing operations |

### 1.2 Architecture Patterns

**Traditional Spring Kafka:**

```java
// Imperative style with explicit template and listeners
@Service
public class OrderService {
    @Autowired
    private KafkaTemplate&lt;String, Order&gt; kafkaTemplate;
    
    public void sendOrder(Order order) {
        kafkaTemplate.send("orders", order.getId(), order);
    }
}
```

## 2. Core Components & Dependencies
### 2.1 Maven Dependencies

Core Spring Kafka:
```
<dependency>
    <groupId>org.springframework.kafka</groupId>
    <artifactId>spring-kafka</artifactId>
    <version>3.3.0</version>
</dependency>

<!-- For Kafka Streams -->
<dependency>
    <groupId>org.apache.kafka</groupId>
    <artifactId>kafka-streams</artifactId>
    <version>3.7.0</version>
</dependency>
```
Spring Boot Starter (Recommended):
```
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.kafka</groupId>
    <artifactId>spring-kafka</artifactId>
</dependency>
Spring Cloud Stream:
```
```
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-stream</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-stream-binder-kafka</artifactId>
</dependency>
```



## 3. Configuration & Setup
### 3.1 Application Properties (YAML)
Basic Configuration:

```yml
spring:
  kafka:
    bootstrap-servers: localhost:9092,localhost:9093
    client-id: ${spring.application.name}
    
    producer:
      key-serializer: org.apache.kafka.common.serialization.StringSerializer
      value-serializer: org.springframework.kafka.support.serializer.JsonSerializer
      acks: all
      retries: 3
      batch-size: 16384
      buffer-memory: 33554432
      properties:
        enable.idempotence: true
        max.in.flight.requests.per.connection: 5
        
    consumer:
      group-id: payment-service-group
      max-poll-records: 500
      max-poll-interval-ms: 300000  # 5 minutes
      session-timeout-ms: 30000
      heartbeat-interval-ms: 10000
      fetch-min-size: 1
      fetch-max-wait-ms: 500
      auto-offset-reset: earliest
      enable-auto-commit: false
      key-deserializer: org.apache.kafka.common.serialization.StringDeserializer
      value-deserializer: org.springframework.kafka.support.serializer.JsonDeserializer
      properties:
        spring.json.trusted.packages: com.example.orders,com.example.payments
        isolation.level: read_committed
        
    listener:
      ack-mode: manual_immediate
      concurrency: 3
      poll-timeout: 3000
      missing-topics-fatal: false

```
### 3.2 Programmatic Configuration
Custom Producer Factory:

```
@Configuration
public class KafkaProducerConfig {
    
    @Bean
    public ProducerFactory<String, Object> producerFactory() {
        Map<String, Object> configProps = new HashMap<>();
        configProps.put(ProducerConfig.BOOTSTRAP_SERVERS_CONFIG, "localhost:9092");
        configProps.put(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG, StringSerializer.class);
        configProps.put(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG, JsonSerializer.class);
        configProps.put(ProducerConfig.ENABLE_IDEMPOTENCE_CONFIG, true);
        configProps.put(ProducerConfig.COMPRESSION_TYPE_CONFIG, "snappy");
        
        return new DefaultKafkaProducerFactory<>(configProps);
    }
    
    @Bean
    public KafkaTemplate<String, Object> kafkaTemplate() {
        return new KafkaTemplate<>(producerFactory());
    }
}
```
Custom Consumer Factory with Error Handling:

```
@Configuration
public class KafkaConsumerConfig {
    
    @Bean
    public ConsumerFactory<String, Object> consumerFactory() {
        Map<String, Object> props = new HashMap<>();
        props.put(ConsumerConfig.BOOTSTRAP_SERVERS_CONFIG, "localhost:9092");
        props.put(ConsumerConfig.GROUP_ID_CONFIG, "inventory-service");
        props.put(ConsumerConfig.AUTO_OFFSET_RESET_CONFIG, "earliest");
        props.put(ConsumerConfig.ENABLE_AUTO_COMMIT_CONFIG, false);
        props.put(ConsumerConfig.MAX_POLL_INTERVAL_MS_CONFIG, 300000);
        
        // JSON deserializer configuration
        JsonDeserializer<Object> deserializer = new JsonDeserializer<>();
        deserializer.addTrustedPackages("com.example.*");
        deserializer.setUseTypeHeaders(false);
        deserializer.setRemoveTypeHeaders(true);
        
        return new DefaultKafkaConsumerFactory<>(
            props, 
            new StringDeserializer(), 
            deserializer
        );
    }
    
    @Bean
    public ConcurrentKafkaListenerContainerFactory<String, Object> kafkaListenerContainerFactory(
            ConsumerFactory<String, Object> consumerFactory) {
        
        ConcurrentKafkaListenerContainerFactory<String, Object> factory = 
            new ConcurrentKafkaListenerContainerFactory<>();
        factory.setConsumerFactory(consumerFactory);
        factory.setConcurrency(3);
        factory.setBatchListener(false);
        factory.getContainerProperties().setAckMode(ContainerProperties.AckMode.MANUAL_IMMEDIATE);
        
        // Error handler
        factory.setCommonErrorHandler(new DefaultErrorHandler(
            new FixedBackOff(1000L, 3L)  // 3 retries with 1 second delay
        ));
        
        return factory;
    }
}

```

## 4. Producer Implementation Patterns
### 4.1 KafkaTemplate Patterns
Basic Send Operations:
```
@Service
@Slf4j
public class OrderProducerService {
    
    @Autowired
    private KafkaTemplate<String, OrderEvent> kafkaTemplate;
    
    // Fire-and-forget (use with caution)
    public void sendOrderAsync(OrderEvent order) {
        kafkaTemplate.send("orders.topic", order.getOrderId(), order);
    }
    
    // Synchronous send with timeout
    public void sendOrderSync(OrderEvent order) throws Exception {
        SendResult<String, OrderEvent> result = kafkaTemplate
            .send("orders.topic", order.getOrderId(), order)
            .get(10, TimeUnit.SECONDS);
        
        log.info("Sent message with offset: {}", result.getRecordMetadata().offset());
    }
    
    // Asynchronous with callback
    public void sendOrderWithCallback(OrderEvent order) {
        ListenableFuture<SendResult<String, OrderEvent>> future = 
            kafkaTemplate.send("orders.topic", order.getOrderId(), order);
        
        future.addCallback(
            result -> log.info("Message sent successfully: {}", result.getRecordMetadata()),
            ex -> log.error("Failed to send message", ex)
        );
    }
}
```
Transaction-Aware Producer:
```
@Service
public class TransactionalOrderService {
    
    @Autowired
    private KafkaTemplate<String, OrderEvent> kafkaTemplate;
    
    @Transactional("kafkaTransactionManager")
    public void processAndSend(Order order) {
        // Database operations within same transaction
        orderRepository.save(order);
        
        // Kafka operations
        kafkaTemplate.send("orders.topic", order.getId(), 
            new OrderEvent(order, EventType.CREATED));
        
        // Both commit together or rollback together
    }
    
    // Explicit transaction boundaries
    public void executeInTransaction(List<Order> orders) {
        kafkaTemplate.executeInTransaction(operations -> {
            orders.forEach(order -> 
                operations.send("orders.topic", order.getId(), order)
            );
            return true;
        });
    }
}
```
### 4.2 Partitioning Strategies
Custom Partitioner:
```
@Component
public class OrderPartitioner implements Partitioner {
    
    @Override
    public int partition(String topic, Object key, byte[] keyBytes, 
                        Object value, byte[] valueBytes, Cluster cluster) {
        List<PartitionInfo> partitions = cluster.partitionsForTopic(topic);
        int numPartitions = partitions.size();
        
        // Ensure same customer orders go to same partition
        String customerId = ((OrderEvent) value).getCustomerId();
        return Math.abs(customerId.hashCode()) % numPartitions;
    }
    
    @Override
    public void close() {}
    
    @Override
    public void configure(Map<String, ?> configs) {}
}
```
Partition-Aware Sending:
```
@Service
public class PartitionedProducer {
    
    @Autowired
    private KafkaTemplate<String, Object> kafkaTemplate;
    
    public void sendToSpecificPartition(String key, Object message, int partition) {
        kafkaTemplate.send("topic", partition, key, message);
    }
    
    // Using default partitioner based on key
    public void sendWithKey(String key, Object message) {
        kafkaTemplate.send("topic", key, message);  // Hash of key determines partition
    }
}
```
## 5. Consumer Implementation Patterns
### 5.1 @KafkaListener Patterns
Basic Listener:
```
@Component
@Slf4j
public class OrderConsumer {
    
    @KafkaListener(
        topics = "orders.topic",
        groupId = "inventory-service",
        containerFactory = "kafkaListenerContainerFactory"
    )
    public void consumeOrder(OrderEvent order) {
        log.info("Received order: {}", order);
        processOrder(order);
    }
}
```
Manual Acknowledgment:
```
@Component
@Slf4j
public class ReliableOrderConsumer {
    
    @KafkaListener(topics = "orders.topic", groupId = "payment-service")
    public void consumeWithManualAck(
            @Payload OrderEvent order,
            @Header(KafkaHeaders.RECEIVED_KEY) String key,
            @Header(KafkaHeaders.RECEIVED_PARTITION) int partition,
            @Header(KafkaHeaders.OFFSET) long offset,
            Acknowledgment acknowledgment) {
        
        try {
            processPayment(order);
            acknowledgment.acknowledge();  // Commit offset
            log.info("Processed order {} from partition {} offset {}", 
                key, partition, offset);
        } catch (Exception e) {
            log.error("Processing failed, not acknowledging", e);
            // Don't acknowledge - message will be redelivered
            throw e;
        }
    }
}
```
Batch Consumption:
```
@Component
@Slf4j
public class BatchOrderConsumer {
    
    @KafkaListener(
        topics = "orders.topic",
        groupId = "analytics-service",
        batch = "true",
        containerFactory = "batchFactory"
    )
    public void consumeBatch(List<OrderEvent> orders, 
                           Acknowledgment acknowledgment) {
        log.info("Received batch of {} orders", orders.size());
        
        try {
            analyticsService.processBulk(orders);
            acknowledgment.acknowledge();
        } catch (Exception e) {
            // Handle partial batch failures
            handleBatchFailure(orders, e);
        }
    }
}
```
### 5.2 Advanced Consumer Patterns
Dynamic Topic Subscription:
```
@Component
public class DynamicTopicConsumer {
    
    @Autowired
    private KafkaListenerEndpointRegistry registry;
    
    // Subscribe to multiple topics dynamically
    @KafkaListener(id = "dynamic-listener", topics = "#{'${app.kafka.topics}'.split(',')}")
    public void listen(String message) {
        // Process message
    }
    
    // Programmatically start/stop listeners
    public void pauseListener() {
        MessageListenerContainer container = 
            registry.getListenerContainer("dynamic-listener");
        container.pause();
    }
}
```
Manual Partition Assignment:
```
@Component
public class PartitionAwareConsumer {
    
    // Assign specific partitions manually
    @KafkaListener(
        topicPartitions = @TopicPartition(
            topic = "compacted-topic",
            partitions = "#{@partitionFinder.partitions('compacted-topic')}",
            partitionOffsets = @PartitionOffset(partition = "*", initialOffset = "0")
        )
    )
    public void listenFromAllPartitions(String payload) {
        // Process all records from all partitions (e.g., cache loading)
    }
}

@Component
public class PartitionFinder {
    
    @Autowired
    private ConsumerFactory<String, String> consumerFactory;
    
    public String[] partitions(String topic) {
        try (Consumer<String, String> consumer = consumerFactory.createConsumer()) {
            return consumer.partitionsFor(topic).stream()
                .map(pi -> String.valueOf(pi.partition()))
                .toArray(String[]::new);
        }
    }
}
```
## 6. Kafka Streams Integration
### 6.1 Basic Streams Configuration
```
@Configuration
@EnableKafkaStreams
public class KafkaStreamsConfig {
    
    @Bean(name = KafkaStreamsDefaultConfiguration.DEFAULT_STREAMS_CONFIG_BEAN_NAME)
    public KafkaStreamsConfiguration kStreamsConfigs() {
        Map<String, Object> props = new HashMap<>();
        props.put(StreamsConfig.APPLICATION_ID_CONFIG, "word-count-app");
        props.put(StreamsConfig.BOOTSTRAP_SERVERS_CONFIG, "localhost:9092");
        props.put(StreamsConfig.DEFAULT_KEY_SERDE_CLASS_CONFIG, Serdes.String().getClass().getName());
        props.put(StreamsConfig.DEFAULT_VALUE_SERDE_CLASS_CONFIG, Serdes.String().getClass().getName());
        props.put(StreamsConfig.COMMIT_INTERVAL_MS_CONFIG, 1000);
        props.put(StreamsConfig.PROCESSING_GUARANTEE_CONFIG, StreamsConfig.EXACTLY_ONCE_V2);
        
        return new KafkaStreamsConfiguration(props);
    }
}
```
### 6.2 Stream Processing Topology
```
@Component
public class WordCountProcessor {
    
    @Autowired
    public void buildTopology(StreamsBuilder streamsBuilder) {
        KStream<String, String> input = streamsBuilder.stream("input-topic");
        
        KTable<String, Long> wordCounts = input
            .flatMapValues(value -> Arrays.asList(value.toLowerCase().split("\\W+")))
            .groupBy((key, value) -> value)
            .count(Materialized.as("word-counts-store"));
        
        wordCounts.toStream().to("output-topic", 
            Produced.with(Serdes.String(), Serdes.Long()));
    }
}
```
### 6.3 Interactive Queries
```
@Service
public class WordCountQueryService {
    
    @Autowired
    private StreamsBuilderFactoryBean factoryBean;
    
    public Long getWordCount(String word) {
        KafkaStreams kafkaStreams = factoryBean.getKafkaStreams();
        ReadOnlyKeyValueStore<String, Long> store = kafkaStreams
            .store(StoreQueryParameters.fromNameAndType(
                "word-counts-store", 
                QueryableStoreTypes.keyValueStore()));
        
        return store.get(word);
    }
}
```
### 6.4 Spring Cloud Stream Kafka Streams
Functional Style (Modern Approach):
```
@SpringBootApplication
public class StreamProcessingApplication {
    
    public static void main(String[] args) {
        SpringApplication.run(StreamProcessingApplication.class, args);
    }
    
    @Bean
    public Function<KStream<String, Order>, KStream<String, ProcessedOrder>> processOrders() {
        return input -> input
            .filter((key, order) -> order.getAmount() > 100)
            .mapValues(this::enrichOrder)
            .selectKey((key, order) -> order.getCustomerId());
    }
    
    @Bean
    public Consumer<KStream<String, Event>> auditLog() {
        return input -> input.foreach((key, event) -> 
            auditService.log(event));
    }
    
    @Bean
    public Supplier<KStream<String, Alert>> generateAlerts() {
        return () -> alertStreamBuilder.buildAlertStream();
    }
}
```

Configuration:

```
spring:
  cloud:
    stream:
      function:
        definition: processOrders;auditLog;generateAlerts
      bindings:
        processOrders-in-0:
          destination: raw-orders
        processOrders-out-0:
          destination: processed-orders
        auditLog-in-0:
          destination: events
        generateAlerts-out-0:
          destination: alerts
      kafka:
        streams:
          binder:
            configuration:
              processing.guarantee: exactly_once_v2
              commit.interval.ms: 1000
```

## 7. Spring Cloud Stream Abstraction
### 7.1 Binder-Based Configuration

```
spring:
  cloud:
    stream:
      kafka:
        binder:
          brokers: localhost:9092
          auto-create-topics: true
          auto-add-partitions: true
          min-partition-count: 3
          replication-factor: 1
      bindings:
        orderInput:
          destination: orders
          group: order-service
          consumer:
            max-attempts: 3
            back-off-initial-interval: 1000
            back-off-max-interval: 10000
        orderOutput:
          destination: processed-orders
          producer:
            partition-key-expression: payload.customerId
            partition-count: 3
```

### 7.2 Functional Programming Model

```
@Component
public class OrderFunctions {
    
    @Bean
    public Function<Order, ProcessedOrder> processOrder() {
        return order -> {
            // Transform logic
            return new ProcessedOrder(order, Status.PROCESSED);
        };
    }
    
    @Bean
    public Function<Flux<Order>, Flux<ProcessedOrder>> processOrderReactive() {
        return flux -> flux
            .map(this::validate)
            .flatMap(this::enrichAsync)
            .map(this::transform);
    }
    
    @Bean
    public Consumer<Order> logOrder() {
        return order -> log.info("Received: {}", order);
    }
}
```

## 8. Transaction Management
### 8.1 Kafka-Only Transactions

```
@Configuration
public class KafkaTransactionConfig {
    
    @Bean
    public ProducerFactory<String, Object> producerFactory() {
        DefaultKafkaProducerFactory<String, Object> factory = 
            new DefaultKafkaProducerFactory<>(producerConfigs());
        factory.setTransactionIdPrefix("tx-order-");
        return factory;
    }
    
    @Bean
    public KafkaTransactionManager<String, Object> kafkaTransactionManager(
            ProducerFactory<String, Object> producerFactory) {
        return new KafkaTransactionManager<>(producerFactory);
    }
}
```

### 8.2 Distributed Transactions (Kafka + Database)
```
@Service
public class DistributedTransactionService {
    
    @Autowired
    private KafkaTemplate<String, OrderEvent> kafkaTemplate;
    
    @Autowired
    private JdbcTemplate jdbcTemplate;
    
    // Chained transaction managers
    @Transactional("chainedTransactionManager")
    public void processOrderWithDatabaseUpdate(Order order) {
        // 1. Update database
        jdbcTemplate.update("INSERT INTO orders (id, status) VALUES (?, ?)",
            order.getId(), "PROCESSING");
        
        // 2. Send to Kafka (both commit together)
        kafkaTemplate.send("orders.topic", order.getId(), 
            new OrderEvent(order, EventType.PROCESSING));
    }
}

@Configuration
public class ChainedTxConfig {
    
    @Bean
    public ChainedKafkaTransactionManager<Object, Object> chainedTransactionManager(
            KafkaTransactionManager<String, Object> kafkaTM,
            DataSourceTransactionManager dataSourceTM) {
        return new ChainedKafkaTransactionManager<>(kafkaTM, dataSourceTM);
    }
    
    @Bean
    public DataSourceTransactionManager dataSourceTransactionManager(DataSource dataSource) {
        return new DataSourceTransactionManager(dataSource);
    }
}
```
### 8.3 Best Practices for Transactions
```
// Idempotent consumer pattern for transactional safety
@Component
public class IdempotentOrderConsumer {
    
    @Autowired
    private ProcessedMessageRepository processedMessageRepository;
    
    @KafkaListener(topics = "orders.topic")
    public void consume(OrderEvent event, @Header(KafkaHeaders.OFFSET) long offset) {
        String messageId = event.getOrderId() + "-" + offset;
        
        // Check if already processed
        if (processedMessageRepository.existsById(messageId)) {
            log.info("Message {} already processed, skipping", messageId);
            return;
        }
        
        // Process message
        processEvent(event);
        
        // Mark as processed
        processedMessageRepository.save(new ProcessedMessage(messageId, Instant.now()));
    }
}
```
## 9. Error Handling & Resilience
### 9.1 Dead Letter Queue (DLQ) Pattern
```
@Configuration
public class ErrorHandlingConfig {
    
    @Bean
    public DefaultErrorHandler errorHandler(KafkaOperations<String, Object> template) {
        // DeadLetterPublishingRecoverer sends failed messages to DLT
        DeadLetterPublishingRecoverer recoverer = new DeadLetterPublishingRecoverer(
            template,
            (record, ex) -> {
                // Custom topic naming strategy
                String topic = record.topic() + ".dlt";
                return new TopicPartition(topic, record.partition());
            }
        );
        
        DefaultErrorHandler handler = new DefaultErrorHandler(
            recoverer,
            new FixedBackOff(2000L, 3L)  // 3 retries, 2s delay
        );
        
        // Classify exceptions - retry on IOException, send to DLT on IllegalArgumentException
        handler.addNotRetryableExceptions(IllegalArgumentException.class);
        handler.addRetryableExceptions(IOException.class);
        
        return handler;
    }
    
    @Bean
    public ConcurrentKafkaListenerContainerFactory<String, Object> kafkaListenerContainerFactory(
            ConsumerFactory<String, Object> consumerFactory,
            DefaultErrorHandler errorHandler) {
        
        ConcurrentKafkaListenerContainerFactory<String, Object> factory = 
            new ConcurrentKafkaListenerContainerFactory<>();
        factory.setConsumerFactory(consumerFactory);
        factory.setCommonErrorHandler(errorHandler);
        return factory;
    }
}
```
### 9.2 Retry with Backoff
```
@Component
public class RetryableConsumer {
    
    @RetryableTopic(
        attempts = "4",
        backoff = @Backoff(delay = 1000, multiplier = 2, maxDelay = 10000),
        include = {IOException.class, TimeoutException.class},
        exclude = {IllegalArgumentException.class},
        dltTopicSuffix = "-dlt",
        retryTopicSuffix = "-retry"
    )
    @KafkaListener(topics = "orders.topic", groupId = "order-service")
    public void consumeWithRetry(OrderEvent order) {
        processOrder(order);
    }
    
    @DltHandler
    public void handleDlt(OrderEvent order, @Header(KafkaHeaders.RECEIVED_TOPIC) String topic) {
        log.error("Message {} from topic {} failed after retries", order, topic);
        alertService.sendAlert(order, topic);
    }
}
```
### 9.3 Circuit Breaker Integration
```
@Component
public class ResilientConsumer {
    
    private final CircuitBreaker circuitBreaker;
    
    public ResilientConsumer() {
        this.circuitBreaker = CircuitBreaker.ofDefaults("kafka-consumer");
    }
    
    @KafkaListener(topics = "external-service-events")
    public void consumeWithCircuitBreaker(Event event) {
        Try.run(() -> circuitBreaker.executeRunnable(() -> callExternalService(event)))
            .recover(throwable -> {
                // Fallback logic - store for later processing
                pendingEventsRepository.save(event);
                return null;
            });
    }
}
```
## 10. Security Implementation
### 10.1 SASL/SSL Configuration

```
spring:
  kafka:
    security:
      protocol: SASL_SSL
    properties:
      sasl.mechanism: SCRAM-SHA-512
      sasl.jaas.config: >
        org.apache.kafka.common.security.scram.ScramLoginModule required
        username="user"
        password="secret";
      ssl.truststore.location: /path/to/truststore.jks
      ssl.truststore.password: ${TRUSTSTORE_PASSWORD}
      ssl.keystore.location: /path/to/keystore.jks
      ssl.keystore.password: ${KEYSTORE_PASSWORD}
      ssl.key.password: ${KEY_PASSWORD}
```

### 10.2 ACLs and Authorization
```
@Configuration
public class KafkaSecurityConfig {
    
    @Bean
    public AdminClient kafkaAdminClient() {
        Properties props = new Properties();
        props.put(AdminClientConfig.BOOTSTRAP_SERVERS_CONFIG, "localhost:9092");
        props.put("security.protocol", "SASL_SSL");
        props.put("sasl.mechanism", "PLAIN");
        // ... security configs
        
        return AdminClient.create(props);
    }
    
    public void createAcls() {
        ResourcePattern resourcePattern = new ResourcePattern(
            ResourceType.TOPIC, 
            "orders.*", 
            PatternType.PREFIXED
        );
        
        AccessControlEntry entry = new AccessControlEntry(
            "User:order-service",
            "*",
            AclOperation.WRITE,
            AclPermissionType.ALLOW
        );
        
        AclBinding binding = new AclBinding(resourcePattern, entry);
        kafkaAdminClient().createAcls(Collections.singleton(binding)).all().get();
    }
}
```
## 11. Testing Strategies
### 11.1 Embedded Kafka Tests
```
@SpringBootTest
@EmbeddedKafka(
    partitions = 1,
    topics = {"orders.topic", "test-topic"},
    brokerProperties = {
        "listeners=PLAINTEXT://localhost:9092",
        "port=9092"
    }
)
class OrderProducerIntegrationTest {
    
    @Autowired
    private OrderProducerService producerService;
    
    @Autowired
    private KafkaTemplate<String, OrderEvent> kafkaTemplate;
    
    @Value("${spring.embedded.kafka.brokers}")
    private String brokerAddresses;
    
    @Test
    void testSendOrder() throws Exception {
        OrderEvent order = new OrderEvent("123", "PRODUCT-1", 100.0);
        
        producerService.sendOrder(order);
        
        // Consume and verify
        Map<String, Object> consumerProps = KafkaTestUtils.consumerProps(
            "test-group", "true", embeddedKafka);
        Consumer<String, OrderEvent> consumer = new DefaultKafkaConsumerFactory<>(
            consumerProps, 
            new StringDeserializer(), 
            new JsonDeserializer<>(OrderEvent.class))
            .createConsumer();
        
        consumer.subscribe(Collections.singleton("orders.topic"));
        
        ConsumerRecord<String, OrderEvent> record = KafkaTestUtils.getSingleRecord(
            consumer, "orders.topic", 5000);
        
        assertThat(record.value().getOrderId()).isEqualTo("123");
    }
}
```

### 11.2 Testcontainers for Integration Testing
```
@Testcontainers
@SpringBootTest
class KafkaIntegrationTest {
    
    @Container
    static KafkaContainer kafka = new KafkaContainer(
        DockerImageName.parse("confluentinc/cp-kafka:7.5.0"))
        .withExposedPorts(9093);
    
    @DynamicPropertySource
    static void kafkaProperties(DynamicPropertyRegistry registry) {
        registry.add("spring.kafka.bootstrap-servers", kafka::getBootstrapServers);
    }
    
    @Test
    void testEndToEndFlow() {
        // Full integration test with real Kafka broker
    }
}
```

### 11.3 Mocking Kafka in Unit Tests

```
@ExtendWith(MockitoExtension.class)
class OrderServiceTest {
    
    @Mock
    private KafkaTemplate<String, OrderEvent> kafkaTemplate;
    
    @InjectMocks
    private OrderService orderService;
    
    @Test
    void testOrderProcessing() {
        when(kafkaTemplate.send(anyString(), anyString(), any()))
            .thenReturn(CompletableFuture.completedFuture(mock(SendResult.class)));
        
        orderService.processOrder(new Order("123", 100.0));
        
        verify(kafkaTemplate).send(
            eq("orders.topic"), 
            eq("123"), 
            any(OrderEvent.class)
        );
    }
}
```

## 12. Monitoring & Observability
### 12.1 Micrometer Metrics

```
management:
  endpoints:
    web:
      exposure:
        include: health,info,prometheus,metrics
  metrics:
    export:
      prometheus:
        enabled: true
    tags:
      application: ${spring.application.name}
```

### 12.2 Custom Health Indicators

```
@Component
public class KafkaHealthIndicator implements HealthIndicator {
    
    @Autowired
    private KafkaTemplate<String, String> kafkaTemplate;
    
    @Override
    public Health health() {
        try {
            // Check connectivity by fetching metadata
            kafkaTemplate.execute(Producer::partitionsFor, "health-check-topic");
            return Health.up()
                .withDetail("bootstrap.servers", "connected")
                .build();
        } catch (Exception e) {
            return Health.down()
                .withException(e)
                .build();
        }
    }
}
```

### 12.3 Consumer Lag Monitoring

```
@Component
public class ConsumerLagMonitor {
    
    @Autowired
    private KafkaListenerEndpointRegistry registry;
    
    @Scheduled(fixedDelay = 60000)
    public void checkConsumerLag() {
        registry.getListenerContainers().forEach(container -> {
            Map<String, Map<Integer, Long>> metrics = container.metrics();
            metrics.forEach((clientId, partitionMetrics) -> {
                partitionMetrics.forEach((partition, lag) -> {
                    if (lag > 1000) {
                        alertService.sendHighLagAlert(clientId, partition, lag);
                    }
                });
            });
        });
    }
}
```
