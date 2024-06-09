# benepik-Task2
Kafka producer to send messages to a Kafka topic & consumer to receive messages from the Kafka 


Step 1: Firstly, we need to define the Kafka Dependencies in'pom.xml' file, Then have to add configuration properties in property file

pom.xml
  <dependency>
    <groupId>org.springframework.kafka</groupId>
    <artifactId>spring-kafka</artifactId>
  </dependency>

Application properties
   kafka:
    order:
      bootstrap-servers: ${KAFKA_RESERVATION_BOOTSTRAP_SERVERS:localhost:9092}
      topic:
        create-order: create-order
      consumer:
        group-id:
          notification: notification
          service: service

Step 2: there are four steps to create a java producer:
* Create producer properties
* Create the producer that send data

 Step 3: there are four steps to create a java consumer:
* Create consumer properties
* Create the consumer that capture data

 Step 4: A rest controller api to hit API:

Execution below :-  

Creating below setting up basic spring boot application entity and config class

@Entity
 public class Order {

   @Getter
   @Setter
   private String orderID;

   @Getter
   @Setter
   private Date dateOfCreation;

   @Getter
   @Setter
   private String content;

  }

// create producer config class 

@Configuration
public class CreateOrderProducerConfig {

    @Value("${spring.kafka.order.bootstrap-servers}")
    private String bootstrapServers;

    @Bean
    public <K, V> ProducerFactory<K, V> createOrderProducerFactory(){
        Map<String,Object> config = new HashMap<>();
        config.put(org.apache.kafka.clients.producer.ProducerConfig.BOOTSTRAP_SERVERS_CONFIG, bootstrapServers);
        config.put(org.apache.kafka.clients.producer.ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG, JsonSerializer.class);
        config.put(org.apache.kafka.clients.producer.ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG, JsonSerializer.class);
        return new DefaultKafkaProducerFactory(config);
    }

    @Bean
    public <K, V> KafkaTemplate<K, V> createOrderKafkaTemplate(){
        return new KafkaTemplate<>(createOrderProducerFactory());
    }
}

// Create order producer class that send data

@Service
public class CreateOrderProducer {

private static final Logger log = LoggerFactory.getLogger(CreateOrderProducer.class);

private final KafkaTemplate<String, Order> createOrderKafkaTemplate;

private final String createOrderTopic;

public CreateOrderProducer(KafkaTemplate<String, Order> createOrderKafkaTemplate,
                              @Value("${spring.kafka.order.topic.create-order}") String createOrderTopic) {
    this.createOrderKafkaTemplate = createOrderKafkaTemplate;
    this.createOrderTopic = createOrderTopic;
}

public boolean sendCreateOrderEvent(Order order) throws ExecutionException, InterruptedException {
    SendResult<String, Order> sendResult = createOrderKafkaTemplate.send(createOrderTopic, order).get();
    log.info("Create order {} event sent via Kafka", order);
    log.info(sendResult.toString());
    return true;
}
}

// Create config class for consumer 


@EnableKafka
@Configuration("NotificationConfiguration")
public class CreateOrderConsumerConfig {

  @Value("${spring.kafka.order.bootstrap-servers}")
  private String bootstrapServers;

  @Value("${spring.kafka.order.consumer.group-id.notification}")
  private String groupId;

  @Bean("NotificationConsumerFactory")
  public ConsumerFactory<String, Order> createOrderConsumerFactory() {
      Map<String, Object> props = new HashMap<>();
      props.put(ConsumerConfig.BOOTSTRAP_SERVERS_CONFIG, bootstrapServers);
      props.put(ConsumerConfig.GROUP_ID_CONFIG, groupId);
      props.put(ConsumerConfig.CLIENT_ID_CONFIG, UUID.randomUUID().toString());
      props.put(ConsumerConfig.KEY_DESERIALIZER_CLASS_CONFIG, StringSerializer.class);
      props.put(ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG, JsonSerializer.class);
      props.put(ConsumerConfig.ENABLE_AUTO_COMMIT_CONFIG, false);

      return new DefaultKafkaConsumerFactory<>(props,new StringDeserializer(),
              new JsonDeserializer<>(Order.class));
  }

  @Bean("NotificationContainerFactory")
  public ConcurrentKafkaListenerContainerFactory<String, Order> createOrderKafkaListenerContainerFactory() {
      ConcurrentKafkaListenerContainerFactory<String, Order> factory =
              new ConcurrentKafkaListenerContainerFactory<>();
      factory.setConsumerFactory(createOrderConsumerFactory());
      factory.getContainerProperties().setAckMode(ContainerProperties.AckMode.MANUAL_IMMEDIATE);
      return factory;
  }
}


// Finally, creating consumer class which will capture data 

 @Service("NotificationService")
  public class CreateOrderConsumer {

      private static final Logger log = LoggerFactory.getLogger(CreateOrderConsumer.class);

      @KafkaListener(topics = "${spring.kafka.order.topic.create-order}", containerFactory="NotificationContainerFactory")
      public void createOrderListener(@Payload Order order, Acknowledgment ack) {
          log.info("Notification service received order {} ", order);
          ack.acknowledge();
      }
  }

// This would be API /api/orders for hitting request from controller class

@RequestMapping("/api/orders")
@RestController
public class OrderController {

    private final CreateOrderProducer createOrderProducer;

    public OrderController(CreateOrderProducer createOrderProducer) {
        this.createOrderProducer = createOrderProducer;
    }

    @PostMapping
    public ResponseEntity<?> createOrder(@RequestBody Order order) throws ExecutionException, InterruptedException {
        createOrderProducer.sendCreateOrderEvent(order);
        return new ResponseEntity<>(HttpStatus.OK);
    }

}


To know the output of the above codes, open the 'kafka-console-consumer' on the CLI using the command:

'kafka-console-consumer -bootstrap-server 127.0.0.1:9092 -topic create-order -group notification'
  
