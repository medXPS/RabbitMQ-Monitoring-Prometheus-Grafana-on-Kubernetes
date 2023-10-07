<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>RabbitMQ Monitoring Guide</title>
</head>
<body>

<h1>RabbitMQ Monitoring Guide</h1>

<p>This guide will walk you through the process of setting up RabbitMQ monitoring using Prometheus and Grafana on Kubernetes.</p>

<h2>Step 1 – Building a RabbitMQ image</h2>

<p>In the first step, we are overriding the Docker image of RabbitMQ. In that case, we need to extend the base image with a tag 3-management and add two plugins. The plugin rabbitmq_prometheus adds Prometheus exporter of core RabbitMQ metrics. The second of them, rabbitmq_tracing allows us to log the payloads of incoming messages. That’s all that we need to define in our Dockerfile, which is visible below.</p>

<pre><code>FROM rabbitmq:3-management
RUN rabbitmq-plugins enable --offline rabbitmq_prometheus rabbitmq_tracing
</code></pre>

<p>Then you just need to build an already defined image. Let’s say its name is piomin/rabbitmq-monitoring. After building I’m pushing it to my remote Docker registry.</p>

<pre><code>$ docker build -t piomin/rabbitmq-monitoring .
$ docker push piomin/rabbitmq-monitoring
</code></pre>

<h2>Step 2 – Deploying RabbitMQ on Kubernetes</h2>

<p>Now, we are going to deploy our custom image of RabbitMQ on Kubernetes. For the purpose of this article, we will run a standalone version of RabbitMQ. In that case, the only thing we need to do is to override some configuration properties in the rabbitmq.conf and enabled_plugins files. First, I’m enabling logging to the console at the debug level. Then I’m also enabling all the required plugins.</p>

<pre><code>apiVersion: v1
kind: ConfigMap
metadata:
  name: rabbitmq
  labels:
    name: rabbitmq
data:
  rabbitmq.conf: |-
    loopback_users.guest = false
    log.console = true
    log.console.level = debug
    log.exchange = true
    log.exchange.level = debug
  enabled_plugins: |-
    [rabbitmq_management,rabbitmq_prometheus,rabbitmq_tracing].
</code></pre>

<p>Both rabbitmq.conf and enabled_plugins files should be placed inside the /etc/rabbitmq directory. Therefore, I’m mounting them inside the volume assigned to the RabbitMQ Deployment. Additionally, we are exposing three ports outside the container. The port 5672 is used in communication with applications through AMQP protocol. The Prometheus plugin exposes metrics on the dedicated port 15692. In order to access Management UI and HTTP endpoints, you should use the 15672 port.</p>

<pre><code>apiVersion: apps/v1
kind: Deployment
metadata:
  name: rabbitmq
  labels:
    app: rabbitmq
spec:
  replicas: 1
  selector:
    matchLabels:
      app: rabbitmq
  template:
    metadata:
      labels:
        app: rabbitmq
    spec:
      containers:
        - name: rabbitmq
          image: piomin/rabbitmq-monitoring:latest
          ports:
            - containerPort: 15672
              name: http
            - containerPort: 5672
              name: amqp
            - containerPort: 15692
              name: prometheus
          volumeMounts:
            - name: rabbitmq-config-map
              mountPath: /etc/rabbitmq/
      volumes:
        - name: rabbitmq-config-map
          configMap:
            name: rabbitmq
</code></pre>

<h2>Step 3 – Building Spring Boot listener application</h2>

<p>Our sample listener application uses Spring Boot AMQP project for integration with RabbitMQ. Thanks to Spring Boot Actuator module it is also exposing metrics including RabbiMQ specific values. It is important to expose them in the format readable by Prometheus.</p>

<pre><code>&lt;dependency&gt;
   &lt;groupId&gt;org.springframework.boot&lt;/groupId&gt;
   &lt;artifactId&gt;spring-boot-starter-amqp&lt;/artifactId&gt;
&lt;/dependency&gt;
&lt;dependency&gt;
   &lt;groupId&gt;org.springframework.boot&lt;/groupId&gt;
   &lt;artifactId&gt;spring-boot-starter-actuator&lt;/artifactId&gt;
&lt;/dependency&gt;
&lt;dependency&gt;
   &lt;groupId&gt;io.micrometer&lt;/groupId&gt;
   &lt;artifactId&gt;micrometer-registry-prometheus&lt;/artifactId&gt;
&lt;/dependency&gt;
</code></pre>

<p>The listener application defines and creates two exchanges. First of them, trx-events-topic, is used for multicast communication. On the other hand, trx-events-direct takes a part in point-to-point communication. Both our sample applications are exchanging messages in JSON format. Therefore we have to override a default Spring Boot AMQP message converter with Jackson2JsonMessageConverter.</p>

<pre><code>@SpringBootApplication
public class ListenerApplication {

   public static void main(String[] args) {
      SpringApplication.run(ListenerApplication.class, args);
   }

   @Bean
   public TopicExchange topic() {
      return new TopicExchange("trx-events-topic");
   }

   @Bean
   public DirectExchange queue() {
      return new DirectExchange("trx-events-direct");
   }

   @Bean
   public MessageConverter jsonMessageConverter() {
      return new Jackson2JsonMessageConverter();
   }
}
</code></pre>

<p>The listener application receives messages from the both topic and direct exchanges. Each running instance of this application is creating a queue binding with the random name. With a direct exchange, only a single queue is receiving incoming messages. On the other hand, all the queues related to a topic exchange are receiving incoming messages.</p>

<pre><code>@Component
@Slf4j
public class ListenerComponent {

   @RabbitListener(bindings = {
      @QueueBinding(
         exchange = @Exchange(type = ExchangeTypes.TOPIC, name = "trx-events-topic"),
         value = @Queue("${topic.queue.name}")
      )
   })
   public void onTopicMessage(SampleMessage message) {
      log.info("Message received: {}", message);
   }

   @RabbitListener(bindings = {
      @QueueBinding(
         exchange = @Exchange(type = ExchangeTypes.DIRECT, name = "trx-events-direct"),
         value = @Queue("${direct.queue.name}")
      )
   })
   public void onDirectMessage(SampleMessage message) {
      log.info("Message received: {}", message);
   }

}
</code></pre>

<p>The name of queues assigned to the topic and direct exchanges is configured inside application.yml file.</p>

<pre><code>topic.queue.name: t-${random.uuid}
direct.queue.name: d-${random.uuid}
</code></pre>

<h2>Step 4 – Building Spring Boot producer application</h2>

<p>The producer application also uses Spring Boot AMQP for integration with RabbitMQ. It sends messages to the exchanges with RabbitTemplate. Similarly to the listener application it formats all the messages as JSON string.</p>

<pre><code>@SpringBootApplication
@EnableScheduling
public class ProducerApplication {

   public static void main(String[] args) {
      SpringApplication.run(ProducerApplication.class, args);
   }

   @Bean
   public RabbitTemplate rabbitTemplate(final ConnectionFactory connectionFactory) {
      final RabbitTemplate rabbitTemplate = new RabbitTemplate(connectionFactory);
      rabbitTemplate.setMessageConverter(producerJackson2MessageConverter());
      return rabbitTemplate;
   }

   @Bean
   public Jackson2JsonMessageConverter producerJackson2MessageConverter() {
      return new Jackson2JsonMessageConverter();
   }

}
</code></pre>

<p>The producer application starts sending messages just after startup. Each message contains id, type and message fields. They are sent to the topic exchange with 5 seconds interval, and to the direct exchange with 2 seconds interval.</p>

<pre><code>@Component
@Slf4j
public class ProducerComponent {

   int index = 1;
   private RabbitTemplate rabbitTemplate;

   ProducerComponent(RabbitTemplate rabbitTemplate) {
      this.rabbitTemplate = rabbitTemplate;
   }

   @Scheduled(fixedRate = 5000)
   public void sendToTopic() {
      SampleMessage msg = new SampleMessage(index++, "abc", "topic");
      rabbitTemplate.convertAndSend("trx-events-topic", null, msg);
      log.info("Sending message: {}", msg);
   }

   @Scheduled(fixedRate = 2000)
   public void sendToDirect() {
      SampleMessage msg = new SampleMessage(index++, "abc", "direct");
      rabbitTemplate.convertAndSend("trx-events-direct", null, msg);
      log.info("Sending message: {}", msg);
   }
   
}
</code></pre>

<h2>Step 5 – Deploying Prometheus and Grafana for RabbitMQ monitoring</h2>

<p>We will use Prometheus for collecting metrics from RabbitMQ, and our both Spring Boot applications. Prometheus detects endpoints with metrics by the Kubernetes Service app label and a HTTP port name. Of course, you can define different search criteria for that. Because Spring Boot and RabbitMQ metrics are defined under different endpoints, we need to define two jobs. On the source code below you can see the ConfigMap that contains the Prometheus configuration file.</p>

<pre><code>apiVersion: v1
kind: ConfigMap
metadata:
  name: prometheus
  labels:
    name: prometheus
data:
  prometheus.yml: |-
    scrape_configs:
      - job_name: 'springboot'
        metrics_path: /actuator/prometheus
        scrape_interval: 5s
        kubernetes_sd_configs:
        - role: endpoints
          namespaces:
            names:
              - default

        relabel_configs:
          - source_labels: [__meta_kubernetes_service_label_app]
            separator: ;
            regex: (producer|listener)
            replacement: $1
            action: keep
          - source_labels: [__meta_kubernetes_endpoint_port_name]
            separator: ;
            regex: http
            replacement: $1
            action: keep
          # ...
      - job_name: 'rabbitmq'
        metrics_path: /metrics
        scrape_interval: 5s
        kubernetes_sd_configs:
        - role: endpoints
          namespaces:
            names:
              - default

        relabel_configs:
          - source_labels: [__meta_kubernetes_service_label_app]
            separator: ;
            regex: rabbitmq
            replacement: $1
            action: keep
          - source_labels: [__meta_kubernetes_endpoint_port_name]
            separator: ;
            regex: prometheus
            replacement: $1
            action: keep
          # ...
</code></pre>

<p>For the full version of Prometheus deployment please refer to the source code. Prometheus tries to discover metric endpoints by the Kubernetes Service label and port name. Let’s take a look on the Service for the listener application.</p>

<pre><code>apiVersion: v1
kind: Service
metadata:
  name: listener-service
  labels:
    app: listener
spec:
  type: ClusterIP
  selector:
    app: listener
  ports:
  - port: 8080
    name: http
</code></pre>

<p>Similarly, we should create Service for RabbitMQ.</p>

<pre><code>apiVersion: v1
kind: Service
metadata:
  name: rabbitmq-service
  labels:
    app: rabbitmq
spec:
  type: NodePort
  selector:
    app: rabbitmq
  ports:
  - port: 15672
    name: http
  - port: 5672
    name: amqp
  - port: 15692
    name: prometheus
</code></pre>

<h2>Step 6 – Deploying stack for RabbitMQ monitoring</h2>

<p>We can finally proceed to the deployment on Kubernetes. In summary, we have five running applications. Three of them, RabbitMQ with the management console, Prometheus, and Grafana are a part of the RabbitMQ monitoring stack. We also have a single instance of the Spring Boot AMQP producer application, and two instances of the Spring Boot AMQP listener application. You can see the final list of pods in the picture below.</p>

<p>If you deploy Spring Boot applications with skaffold dev --port-forward command, you can easily access them on the local port. Other applications can be accessed via Kubernetes Service NodePort.</p>

<h2>Step 7 – RabbitMQ monitoring with Prometheus metrics</h2>

<p>We can easily verify a list of metrics generated by Spring Boot applications by calling /actuator/prometheus endpoint. First, let’s take a look at the metrics returned by the listener application.</p>

<pre><code>rabbitmq_not_acknowledged_published_total{name="rabbit",} 0.0
rabbitmq_unrouted_published_total{name="rabbit",} 0.0
rabbitmq_channels{name="rabbit",} 2.0
rabbitmq_consumed_total{name="rabbit",} 2432.0
rabbitmq_connections{name="rabbit",} 1.0
rabbitmq_acknowledged_total{name="rabbit",} 2432.0
spring_rabbitmq_listener_seconds_max{exception="none",listener_id="org.springframework.amqp.rabbit.RabbitListenerEndpointContainer#1",queue="d-ea28bd07-929d-4928-8d2c-5dceeec9950a",result="success",} 0.0025406
spring_rabbitmq_listener_seconds_max{exception="none",listener_id="org.springframework.amqp.rabbit.RabbitListenerEndpointContainer#0",queue="t-e8990fe4-7d8c-4a2c-96d7-fff4fe503265",result="success",} 0.0024175
spring_rabbitmq_listener_seconds_count{exception="none",listener_id="org.springframework.amqp.rabbit.RabbitListenerEndpointContainer#1",queue="d-ea28bd07-929d-4928-8d2c-5dceeec9950a",result="success",} 1712.0
spring_rabbitmq_listener_seconds_sum{exception="none",listener_id="org.springframework.amqp.rabbit.RabbitListenerEndpointContainer#1",queue="d-ea28bd07-929d-4928-8d2c-5dceeec9950a",result="success",} 0.992886413
spring_rabbitmq_listener_seconds_count{exception="none",listener_id="org.springframework.amqp.rabbit.RabbitListenerEndpointContainer#0",queue="t-e8990fe4-7d8c-4a2c-96d7-fff4fe503265",result="success",} 720.0
spring_rabbitmq_listener_seconds_sum{exception="none",listener_id="org.springframework.amqp.rabbit.RabbitListenerEndpointContainer#0",queue="t-e8990fe4-7d8c-4a2c-96d7-fff4fe503265",result="success",} 0.598468801
rabbitmq_acknowledged_published_total{name="rabbit",} 0.0
rabbitmq_failed_to_publish_total{name="rabbit",} 0.0
rabbitmq_rejected_total{name="rabbit",} 0.0
rabbitmq_published_total{name="rabbit",} 0.0
</code></pre>

<p>Similarly, we can verify the list of metrics from the producer application. In contrast to the listener application, it is generating rabbitmq_published_total instead of rabbitmq_consumed_total.</p>

<pre><code>rabbitmq_acknowledged_published_total{name="rabbit",} 0.0
rabbitmq_unrouted_published_total{name="rabbit",} 0.0
rabbitmq_acknowledged_total{name="rabbit",} 0.0
rabbitmq_rejected_total{name="rabbit",} 0.0
rabbitmq_connections{name="rabbit",} 1.0
rabbitmq_not_acknowledged_published_total{name="rabbit",} 0.0
rabbitmq_consumed_total{name="rabbit",} 0.0
rabbitmq_failed_to_publish_total{name="rabbit",} 0.0
rabbitmq_published_total{name="rabbit",} 2553.0
rabbitmq_channels{name="rabbit",} 1.0
</code></pre>

<p>The list of metrics generated by the RabbitMQ Prometheus plugin is pretty impressive. I decided to use only some of them.</p>

<pre><code>rabbitmq_channel_consumers 6
rabbitmq_channel_messages_published_total 2926
rabbitmq_channel_messages_delivered_total 2022
rabbitmq_channel_messages_acked_total 5922
rabbitmq_connections_opened_total 9
rabbitmq_connection_incoming_bytes_total 700878
rabbitmq_connection_outgoing_bytes_total 1388158
rabbitmq_connection_channels 5
rabbitmq_queue_messages 5917
rabbitmq_queue_consumers 6
rabbitmq_queues 10
</code></pre>

<p>We can visualize all the metrics on the Grafana dashboard. Grafana is using Prometheus as a data source. To repeat, Prometheus collects data from the Spring Boot applications endpoints and RabbitMQ.</p>

<p>but indicate only file name instead of its code</p>

</body>
</html>
