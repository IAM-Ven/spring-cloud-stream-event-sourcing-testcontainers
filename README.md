# spring-cloud-stream-event-sourcing-testcontainers

The goal of this project is to create a [`Spring Boot`](https://docs.spring.io/spring-boot/docs/current/reference/htmlsingle/) application that handles `users` using [`Event Sourcing`](https://martinfowler.com/eaaDev/EventSourcing.html). So, besides the traditional create/update/delete, whenever a user is created, updated or deleted, an event informing this change is sent to [`Kafka`](https://kafka.apache.org). Furthermore, we will implement another `Spring Boot` application that listens to those events and saves them in [`Cassandra`](https://cassandra.apache.org). Finally, we will use [`Testcontainers`](https://www.testcontainers.org) to run the integration tests.

> **Note:** In [`kubernetes-environment`](https://github.com/ivangfr/kubernetes-environment/tree/master/user-event-sourcing-kafka) repository, it is shown how to deploy this project in `Kubernetes` (`Minikube`)

## Project Architecture

![project-diagram](images/project-diagram.png)

## Applications

- ### user-service

  `Spring Boot` Web Java application responsible for handling users. The user information is stored in [`MySQL`](https://www.mysql.com). Once a user is created, updated or deleted, an event is sent to `Kafka`.
  
  ![user-service](images/user-service-swagger.png)

  - **Serialization format**

    `user-service` can use [`JSON`](https://www.json.org) or [`Avro`](https://avro.apache.org) format to serialize data to the `binary` format used by `Kafka`. If we choose `Avro`, both services will benefit by the [`Schema Registry`](https://docs.confluent.io/current/schema-registry/docs/index.html) that is running as Docker container. The serialization format to be used is defined by the value set to the environment variable `SPRING_PROFILES_ACTIVE`.
  
    | Configuration                    | Format |
    | -------------------------------- | ------ |
    | `SPRING_PROFILES_ACTIVE=default` | `JSON` |
    | `SPRING_PROFILES_ACTIVE=avro`    | `Avro` |

- ### event-service

  `Spring Boot` Web Java application responsible for listening events from `Kafka` and saving them in `Cassandra`.

  ![event-service](images/event-service-swagger.png)

  - **Deserialization**
  
    Differently from `user-service`, `event-service` has no specific Spring profile to select the deserialization format. [`Spring Cloud Stream`](https://docs.spring.io/spring-cloud-stream/docs/current/reference/htmlsingle) provides a stack of `MessageConverters` that handle the conversion of many types of content-types, including `application/json`. Besides, as `event-service` has `SchemaRegistryClient` bean registered, `Spring Cloud Stream` auto configures an Apache Avro message converter for schema management.
    
    In order to handle different content-types, `Spring Cloud Stream` has a _"content-type negotiation and transformation"_ strategy (more [here](https://docs.spring.io/spring-cloud-stream/docs/current/reference/htmlsingle/#content-type-management)). The precedence orders are: first, content-type present in the message header; second, content-type defined in the binding; and finally, content-type is `application/json` (default).
    
    The producer (in the case `user-service`) always sets the content-type in the message header. The content-type can be `application/json` or `application/*+avro`, depending on with which `SPRING_PROFILES_ACTIVE` the `user-service` is started.
  
  - **Java classes from Avro Schema**
  
    Run the following command in `spring-cloud-stream-event-sourcing-testcontainers` root folder. It will re-generate the Java classes from the Avro schema present at `event-service/src/main/resources/avro`.
    ```
    ./gradlew event-service:generateAvro
    ```
  
## Prerequisites

- [`Java 11+`](https://www.oracle.com/java/technologies/javase-jdk11-downloads.html)
- [`Docker`](https://www.docker.com/)
- [`Docker-Compose`](https://docs.docker.com/compose/install/)

## Start Environment

- In a terminal and inside `spring-cloud-stream-event-sourcing-testcontainers` root folder run
  ```
  docker-compose up -d
  ```

- Wait until all containers are `Up (healthy)`. You can check by running the following command
  ```
  docker-compose ps
  ```

## Running Applications with Gradle

- **user-service**

  - In a terminal, make sure you are inside `spring-cloud-stream-event-sourcing-testcontainers` root folder
  
  - In order to run the application, you can pick between `JSON` or `Avro`
    - Using `JSON`
      ```
      ./gradlew user-service:clean user-service:bootRun --args='--server.port=9080'
      ```
    - Using `Avro`
      ```
      ./gradlew user-service:clean user-service:bootRun --args='--server.port=9080 --spring.profiles.active=avro'
      ```

- **event-service**

  - In a new terminal, make sure you are inside `spring-cloud-stream-event-sourcing-testcontainers` root folder
  
  - Run the following command
    ```
    ./gradlew event-service:clean event-service:bootRun --args='--server.port=9081'
    ```

## Running Applications as Docker containers

### Build Application's Docker Image

- In a terminal, make sure you are inside `spring-cloud-stream-event-sourcing-testcontainers` root folder

- Run the following script to build the Docker images
  - JVM 
    ```
    ./build-apps.sh
    ```
  - Native
    ```
    ./build-apps.sh native
    ```

### Application's Environment Variables
   
- **user-service**

  | Environment Variable   | Description                                                                          |
  | ---------------------- | ------------------------------------------------------------------------------------ |
  | `MYSQL_HOST`           | Specify host of the `MySQL` database to use (default `localhost`)                    |
  | `MYSQL_PORT`           | Specify port of the `MySQL` database to use (default `3306`)                         |
  | `KAFKA_HOST`           | Specify host of the `Kafka` message broker to use (default `localhost`)              |
  | `KAFKA_PORT`           | Specify port of the `Kafka` message broker to use (default `29092`)                  |
  | `SCHEMA_REGISTRY_HOST` | Specify host of the `Schema Registry` to use (default `localhost`)                   |
  | `SCHEMA_REGISTRY_PORT` | Specify port of the `Schema Registry` to use (default `8081`)                        |
  | `ZIPKIN_HOST`          | Specify host of the `Zipkin` distributed tracing system to use (default `localhost`) |
  | `ZIPKIN_PORT`          | Specify port of the `Zipkin` distributed tracing system to use (default `9411`)      |

- **event-service**

  | Environment Variable   | Description                                                                          |
  | ---------------------- | ------------------------------------------------------------------------------------ |
  | `CASSANDRA_HOST`       | Specify host of the `Cassandra` database to use (default `localhost`)                |
  | `CASSANDRA_PORT`       | Specify port of the `Cassandra` database to use (default `9042`)                     |
  | `KAFKA_HOST`           | Specify host of the `Kafka` message broker to use (default `localhost`)              |
  | `KAFKA_PORT`           | Specify port of the `Kafka` message broker to use (default `29092`)                  |
  | `SCHEMA_REGISTRY_HOST` | Specify host of the `Schema Registry` to use (default `localhost`)                   |
  | `SCHEMA_REGISTRY_PORT` | Specify port of the `Schema Registry` to use (default `8081`)                        |
  | `ZIPKIN_HOST`          | Specify host of the `Zipkin` distributed tracing system to use (default `localhost`) |
  | `ZIPKIN_PORT`          | Specify port of the `Zipkin` distributed tracing system to use (default `9411`)      |

### Start Application's Docker Container

- In a terminal, make sure you are inside `spring-cloud-stream-event-sourcing-testcontainers` root folder

- In order to run the application's docker container, you can pick between `JSON` or `Avro`
  - Using `JSON`
    ```
    ./start-apps.sh
    ```
  - Using `Avro`
    ```
    ./start-apps.sh avro
    ```

## Applications URLs

| Application   | URL                                   |
| ------------- | ------------------------------------- |
| user-service  | http://localhost:9080/swagger-ui.html |
| event-service | http://localhost:9081/swagger-ui.html |

## Playing around

1. Create a user
   ```
   curl -i -X POST localhost:9080/api/users \
     -H  "Content-Type: application/json" \
     -d '{"email":"ivan.franchin@test.com","fullName":"Ivan Franchin","active":true}'
   ```

1. Check whether the event related to the user creation was received by `event-service`
   ```
   curl -i localhost:9081/api/events/users/1
   ```

1. You can check me message trace using [`Zipkin`](https://zipkin.io) http://localhost:9411. The picture below shows an example 

   ![zipkin](images/zipkin.png)

1. Access `user-service` and `event-service` Swagger website and create new users and/or update/delete existing

## Useful Commands & Links

- **MySQL**
  ```
  docker exec -it mysql mysql -uroot -psecret --database userdb
  select * from users;
  ```

- **Cassandra**
  ```
  docker exec -it cassandra cqlsh
  USE mycompany;
  SELECT * FROM user_events;
  ```

- **Zipkin**

  `Zipkin` can be accessed at http://localhost:9411

- **Kafka Topics UI**

  `Kafka Topics UI` can be accessed at http://localhost:8085

  ![kafka-topics-ui](images/kafka-topics-ui.png)

- **Schema Registry UI**

  `Schema Registry UI` can be accessed at http://localhost:8001

  ![schema-registry-ui](images/schema-registry-ui.png)

- **Kafka Manager**

  `Kafka Manager` can be accessed at http://localhost:9000

  _Configuration_

  - First, you must create a new cluster. Click on `Cluster` (dropdown button on the header) and then on `Add Cluster`
  - Type the name of your cluster in `Cluster Name` field, for example: `MyCluster`
  - Type `zookeeper:2181` in `Cluster Zookeeper Hosts` field
  - Enable checkbox `Poll consumer information (Not recommended for large # of consumers if ZK is used for offsets tracking on older Kafka versions)`
  - Click on `Save` button at the bottom of the page.

  The image below shows the topics present in Kafka, including the topic `com.mycompany.userservice.user` with `3`
partitions.

  ![kafka-manager](images/kafka-manager.png)

## Shutdown

- Stop applications
  - If they were started with `Gradle`, go to the terminals where they are running and press `Ctrl+C`
  - If they were started as a Docker container, run the script below
    ```
    ./stop-apps.sh
    ```

- To stop and remove docker-compose containers, networks and volumes, make sure you are inside `spring-cloud-stream-event-sourcing-testcontainers` root folder and run
  ```
  docker-compose down -v
  ```

## Running tests

- **event-service**

  - Run the command below to start the tests
    ```
    ./gradlew event-service:clean event-service:cleanTest event-service:test
    ```

- **user-service**

  - During integration tests, [`Testcontainers`](https://www.testcontainers.org/) will start automatically `Zookeeper`, `Kafka`, `MySQL`, `Cassandra` and `event-service` containers before the tests begin and shuts them down when the tests finish.
    > **Note:** Make sure you have an updated `event-service` docker image

  - Run the command below to start the tests
    - Using `JSON`
      ```
      ./gradlew user-service:clean user-service:cleanTest user-service:test
      ```
    - Using `Avro`
      > **Warning:** app starts with correct profile. However, messages are sent in json format.
      ```
      SPRING_PROFILES_ACTIVE=avro ./gradlew user-service:clean user-service:cleanTest user-service:test
      ```

## Issues

- When building the Docker native image of `event-service` and `user-service`, it's throwing the following exception
  ```
      [creator]     Fatal error:java.lang.IllegalStateException: java.lang.IllegalStateException: ERROR: in 'org.springframework.cloud.schema.registry.avro.AvroMessageConverterAutoConfiguration' these methods are directly invoking methods marked @Bean: [avroSchemaMessageConverter] - due to the enforced proxyBeanMethods=false for components in a native-image, please consider refactoring to use instance injection. If you are confident this is not going to affect your application, you may turn this check off using -Dspring.native.verify=false.
      [creator]           at java.base/jdk.internal.reflect.NativeConstructorAccessorImpl.newInstance0(Native Method)
      [creator]           at java.base/jdk.internal.reflect.NativeConstructorAccessorImpl.newInstance(NativeConstructorAccessorImpl.java:62)
      [creator]           at java.base/jdk.internal.reflect.DelegatingConstructorAccessorImpl.newInstance(DelegatingConstructorAccessorImpl.java:45)
      [creator]           at java.base/java.lang.reflect.Constructor.newInstance(Constructor.java:490)
      [creator]           at java.base/java.util.concurrent.ForkJoinTask.getThrowableException(ForkJoinTask.java:600)
      [creator]           at java.base/java.util.concurrent.ForkJoinTask.get(ForkJoinTask.java:1006)
      [creator]           at com.oracle.svm.hosted.NativeImageGenerator.run(NativeImageGenerator.java:483)
      [creator]           at com.oracle.svm.hosted.NativeImageGeneratorRunner.buildImage(NativeImageGeneratorRunner.java:350)
      [creator]           at com.oracle.svm.hosted.NativeImageGeneratorRunner.build(NativeImageGeneratorRunner.java:509)
      [creator]           at com.oracle.svm.hosted.NativeImageGeneratorRunner.main(NativeImageGeneratorRunner.java:115)
      [creator]           at com.oracle.svm.hosted.NativeImageGeneratorRunner$JDK9Plus.main(NativeImageGeneratorRunner.java:541)
      [creator]     Caused by: java.lang.IllegalStateException: ERROR: in 'org.springframework.cloud.schema.registry.avro.AvroMessageConverterAutoConfiguration' these methods are directly invoking methods marked @Bean: [avroSchemaMessageConverter] - due to the enforced proxyBeanMethods=false for components in a native-image, please consider refactoring to use instance injection. If you are confident this is not going to affect your application, you may turn this check off using -Dspring.native.verify=false.
      [creator]           at org.springframework.graalvm.type.Type.verifyComponent(Type.java:2273)
      [creator]           at org.springframework.graalvm.support.ResourcesHandler.processType(ResourcesHandler.java:1282)
      [creator]           at org.springframework.graalvm.support.ResourcesHandler.processType(ResourcesHandler.java:960)
      [creator]           at org.springframework.graalvm.support.ResourcesHandler.checkAndRegisterConfigurationType(ResourcesHandler.java:950)
      [creator]           at org.springframework.graalvm.support.ResourcesHandler.processSpringFactory(ResourcesHandler.java:849)
      [creator]           at org.springframework.graalvm.support.ResourcesHandler.processSpringFactories(ResourcesHandler.java:714)
      [creator]           at org.springframework.graalvm.support.ResourcesHandler.register(ResourcesHandler.java:130)
      [creator]           at org.springframework.graalvm.support.SpringFeature.beforeAnalysis(SpringFeature.java:107)
      [creator]           at com.oracle.svm.hosted.NativeImageGenerator.lambda$runPointsToAnalysis$7(NativeImageGenerator.java:696)
      [creator]           at com.oracle.svm.hosted.FeatureHandler.forEachFeature(FeatureHandler.java:70)
      [creator]           at com.oracle.svm.hosted.NativeImageGenerator.runPointsToAnalysis(NativeImageGenerator.java:696)
      [creator]           at com.oracle.svm.hosted.NativeImageGenerator.doRun(NativeImageGenerator.java:558)
      [creator]           at com.oracle.svm.hosted.NativeImageGenerator.lambda$run$0(NativeImageGenerator.java:471)
      [creator]           at java.base/java.util.concurrent.ForkJoinTask$AdaptedRunnableAction.exec(ForkJoinTask.java:1407)
      [creator]           at java.base/java.util.concurrent.ForkJoinTask.doExec(ForkJoinTask.java:290)
      [creator]           at java.base/java.util.concurrent.ForkJoinPool$WorkQueue.topLevelExec(ForkJoinPool.java:1020)
      [creator]           at java.base/java.util.concurrent.ForkJoinPool.scan(ForkJoinPool.java:1656)
      [creator]           at java.base/java.util.concurrent.ForkJoinPool.runWorker(ForkJoinPool.java:1594)
      [creator]           at java.base/java.util.concurrent.ForkJoinWorkerThread.run(ForkJoinWorkerThread.java:183)
      [creator]     Error: Image build request failed with exit status 1
      [creator]     unable to invoke layer creator
      [creator]     unable to contribute native-image layer
      [creator]     error running build
      [creator]     exit status 1
      [creator]     ERROR: failed to build: exit status 1
  ```

- When using the dependency `org.springframework.cloud:spring-cloud-stream:test-binder@test-jar` instead of `org.springframework.cloud:spring-cloud-stream-test-support`, it is throwing the following exception
  ```
   * What went wrong:
   Execution failed for task ':event-service:compileTestJava'.
   > Could not resolve all files for configuration ':event-service:testCompileClasspath'.
      > Could not find org.springframework.cloud:spring-cloud-stream:test-binder.
        Required by:
            project :event-service
      > Could not find org.springframework.cloud:spring-cloud-stream:test-binder.
        Required by:
            project :event-service > org.springframework.cloud:spring-cloud-starter-stream-kafka:3.1.0-M4 > org.springframework.cloud:spring-cloud-stream-binder-kafka:3.1.0-M4
            project :event-service > org.springframework.cloud:spring-cloud-starter-stream-kafka:3.1.0-M4 > org.springframework.cloud:spring-cloud-stream-binder-kafka:3.1.0-M4 > org.springframework.cloud:spring-cloud-stream-binder-kafka-core:3.1.0-M4
  ```

## References

- https://docs.spring.io/spring-cloud-stream/docs/current/reference/html/spring-cloud-stream.html
