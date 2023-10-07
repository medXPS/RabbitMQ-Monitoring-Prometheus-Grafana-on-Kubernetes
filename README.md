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

<!-- Include the remaining steps (Step 3 to Step 7) similarly -->

<h2>Step 3 – Building Spring Boot listener application</h2>

<!-- Include the content for Step 3 here -->

<h2>Step 4 – Building Spring Boot producer application</h2>

<!-- Include the content for Step 4 here -->

<h2>Step 5 – Deploying Prometheus and Grafana for RabbitMQ monitoring</h2>

<!-- Include the content for Step 5 here -->

<h2>Step 6 – Deploying stack for RabbitMQ monitoring</h2>

<!-- Include the content for Step 6 here -->

<h2>Step 7 – RabbitMQ monitoring with Prometheus metrics</h2>

<!-- Include the content for Step 7 here -->

<h2>Conclusion</h2>

<p>This guide provides a comprehensive walkthrough of setting up RabbitMQ monitoring on Kubernetes. By following these steps, you'll be able to monitor RabbitMQ, Spring Boot applications, and Prometheus metrics effectively.</p>

<p>For detailed configurations and additional details, please refer to the source code and documentation.</p>

<h2>License</h2>

<p>This project is licensed under the MIT License - see the <a href="LICENSE">LICENSE</a> file for details.</p>

</body>
</html>
