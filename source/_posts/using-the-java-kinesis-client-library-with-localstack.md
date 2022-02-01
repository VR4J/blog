---
title: Using the Java Kinesis Client Library (KCL) with localstack
icon: 💨
image: /images/using-the-java-kinesis-client-library-kcl-with-localstack-social.jpg
tags:
    - Java 17
    - AWS
    - Kinesis
    - DynamoDB
    - LocalStack
    - Spring Boot
excerpt: Read or heard alot about Kinesis Data Streams, and eager to know how to actually use it? Come along and join me in this magical adventure of using the Kinesis Client Library and running it locally with LocalStack!
---

Wait wait, hear me out: Consuming a multi-sharded kinesis stream by implementing the Java Kinesis Client Library, and to top it off, use [localstack](https://localstack.cloud) to run our code locally. Interested? Great, let's go!

Only interested in the result? Here is the [Github Repository](https://github.com/VR4J/java-kinesis-client-consumer).

## Kinesis Data Streams
Kinesis Data Streams is one of the many, many, managed services provided by Amazon Web Services (AWS). Kinesis streams can be _sharded_ based on a partition key, allowing multiple consumers to each consume a specific shard in parallel.

Sharding is a way of scaling the consumers on a stream, while still maintaining the order of the messages based on the partition key.
<img src="/images/sharding-principle.png" />

With this example, you can see that message A-1 can be handled in parallel of message B-1, but messages A-1 and A-2 are always processed in order by the same consumer.

## Configuring localstack
Usually we would setup the Kinesis Stream using CloudFormation, Terraform, or the AWS Console itself, but since we want to be able to test our code locally we will use [localstack](https://localstack.cloud) instead.

Localstack is "a fully functional local cloud stack", which means it gives us a local implementation of almost every AWS service. Luckily we don't need _all_ the services provided by Amazon, but only Kinesis (obviously), DynamoDB and Cloudwatch.

We need DynamoDB as the Kinesis Client Library (KCL) uses a DynamoDB table to store the checkpoints of all shards. Checkpoints can be considered the state of the client, containing which application has the lease on which shard, and at which sequencenumber the consumer is currently processing the data.

We need CloudWatch as the Kinesis Client Library (KCL) provides metrics to monitor our application.

### Setup
```sh
$ pip install localstack
$ pip install awscli-local
```

### Running
```sh 
$ SERVICES=kinesis,dynambodb,cloudwatch localstack start -d 

     __                     _______ __             __
    / /   ____  _________ _/ / ___// /_____ ______/ /__
   / /   / __ \/ ___/ __ `/ /\__ \/ __/ __ `/ ___/ //_/
  / /___/ /_/ / /__/ /_/ / /___/ / /_/ /_/ / /__/ ,<
 /_____/\____/\___/\__,_/_//____/\__/\__,_/\___/_/|_|

 💻 LocalStack CLI 0.13.3.3

[20:22:20] starting LocalStack in Docker mode 🐳
[20:22:21] detaching
```

Awesome, we can now interact with our local AWS services by using the awslocal cli and create our Kinesis stream.

```sh
$ awslocal kinesis create-stream --stream-name some-data-stream --shard-count 2
```

And that's all we need, we have our localstack environment and we have our Kinesis data stream with two shards, time to get our hands dirty and build some actual code!

## Setting up the Kinesis Client Library

### Dependencies
Let's start with the **build.gradle** file which lists all the dependencies we will need.

```groovy
plugins {
	id 'org.springframework.boot' version '2.6.3'
	id 'io.spring.dependency-management' version '1.0.11.RELEASE'
	id 'java'
}

group = 'com.vreijsen'
version = '0.0.1-SNAPSHOT'
sourceCompatibility = '17'

configurations {
	compileOnly {
		extendsFrom annotationProcessor
	}
}

repositories {
	mavenCentral()
}

tasks.withType(JavaCompile) {
	options.compilerArgs += "--enable-preview"
}

dependencies {
	annotationProcessor("org.springframework.boot:spring-boot-configuration-processor")

	implementation("org.springframework.boot:spring-boot-starter-webflux")
	implementation('org.springframework:spring-context')

	implementation("javax.servlet:javax.servlet-api:4.0.1")

	// Kinesis Client Library
	implementation('software.amazon.kinesis:amazon-kinesis-client:2.3.10')

	compileOnly 'org.projectlombok:lombok:1.18.20'
	annotationProcessor 'org.projectlombok:lombok:1.18.20'
	testImplementation 'org.springframework.boot:spring-boot-starter-test'
}

tasks.named('test') {
	useJUnitPlatform()
}
```

### Configuration
We'll have to instruct our application which Kinesis Data Stream to consume, and in which region this stream lives. We can default the region to `US_EAST_1` since that is the default region used by localstack.

```java
package com.vreijsen.consumer.configuration;

@Data
@Configuration
@ConfigurationProperties(prefix = "kinesis")
public class KinesisConfigurationProperties {

    String stream;
    Region region = Region.US_EAST_1;
}
```
Our **application.yaml** can then contain the following.
```yaml
kinesis:
  stream: "some-data-stream"
```

Since we'll be using three AWS services, we have to define a *client* bean for each one of them.

```java
@Profile("localstack")
@Configuration
@RequiredArgsConstructor
public class LocalStackClientConfiguration {

    private static final URI LOCALSTACK_URI = URI.create("http://localhost:4566");

    private final KinesisConfigurationProperties properties;

    @Bean
    public KinesisAsyncClient kinesisAsyncClient() {
        return KinesisClientUtil.createKinesisAsyncClient(
                KinesisAsyncClient.builder()
                        .endpointOverride(LOCALSTACK_URI)
                        .region(properties.getRegion())
        );
    }

    @Bean
    public DynamoDbAsyncClient dynamoAsyncClient() {
        return DynamoDbAsyncClient.builder()
                .endpointOverride(LOCALSTACK_URI)
                .region(properties.getRegion())
                .build();
    }

    @Bean
    public CloudWatchAsyncClient cloudWatchAsyncClient() {
        return CloudWatchAsyncClient.builder()
                .endpointOverride(LOCALSTACK_URI)
                .region(properties.getRegion())
                .build();
    }
}
```
You can see we annotated the class with `@Profile("localstack")` as it overrides the endpoints on the clients to point to our localstack environment running on our own machine.

We can define the same class but without the endpoint overrides and annotate it with `@Profile("!localstack")` to connect to the actual AWS cloud when not running with the localstack profile.

### Kinesis Scheduler
The **KinesisScheduler** is the heart of the library and will do all our Kinesis magic. We can provide the clients that we just setup, but most importantly provide a **ShardRecordProcessor** which will be created for every shard the application holds the lease of. 

When implementing the **ShardRecordProcessor** we have a few methods that we can override to adjust the behaviour to our specific needs. 

The most important method to override is the `processRecords(ProcessRecordsInput input)` as it is the entry point for all data coming in from that shard. For our demo, we'll just log every record that comes in.

```java
@Slf4j
public class KinesisRecordProcessor implements ShardRecordProcessor {

    @Override
    /* Called when initializing the record processor; can be used to set some MDC properties before receiving data. */
    public void initialize(InitializationInput input) { }

    @Override
    public void processRecords(ProcessRecordsInput input) {
        input.records().forEach(record -> log.info("Received Kinesis message: {}.", record));
    }

    @Override
    /* Called when the lease has been lost to another consumer; can be used to remove some MDC properties. */
    public void leaseLost(LeaseLostInput input) { }

    @Override
    /* Called when the last message of the shard has been processed, and we need to persist our checkpoint. */
    public void shardEnded(ShardEndedInput input) {
        try {
            input.checkpointer().checkpoint();
        } catch (InvalidStateException | ShutdownException e) {
            log.error("Exception while checkpointing at shard end. Giving up.", e);
        }
    }

    @Override
    /* Called when the app is shutting down, so we need to persist our current checkpoint. */
    public void shutdownRequested(ShutdownRequestedInput input) {
        try {
            input.checkpointer().checkpoint();
        } catch (InvalidStateException | ShutdownException e) {
            log.error("Exception while checkpointing at shutdown. Giving up.", e);
        }
    }
}
```

Alright now that we have a **ShardRecordProcessor** we can initialize the Kinesis Scheduler.

```java
@Configuration
@RequiredArgsConstructor
public class KinesisConfiguration {

    private final KinesisConfigurationProperties properties;

    @Bean
    public Scheduler scheduler(KinesisAsyncClient kinesis, DynamoDbAsyncClient dynamodb, CloudWatchAsyncClient cloudwatch) {
        ConfigsBuilder configs = new ConfigsBuilder(properties.getStream(), properties.getStream(), kinesis, dynamodb, cloudwatch,
                UUID.randomUUID().toString(),
                KinesisRecordProcessor::new
        );

        return new Scheduler(
                configs.checkpointConfig(),
                configs.coordinatorConfig(),
                configs.leaseManagementConfig(),
                configs.lifecycleConfig(),
                configs.metricsConfig(),
                configs.processorConfig(),
                configs.retrievalConfig().retrievalSpecificConfig(
                        new PollingConfig(properties.getStream(), kinesis)
                )
        );
    }
}
```

With this scheduler we can now create our deamon thread and start consuming!

```java
@Component
@RequiredArgsConstructor
public class KinesisRunner {

    private final Scheduler scheduler;

    @PostConstruct
    public void run() {
        Thread schedulerThread = new Thread(scheduler);
        schedulerThread.setDaemon(true);
        schedulerThread.start();
    }
}
```

Ofcourse to be able to consume data we'll first need to send some data to the Kinesis stream. Keep in mind the `Data` property should be Base64 encoded.

```sh 
awslocal kinesis put-records --stream-name some-data-stream --records Data=SGVsbG8gV29ybGQ=,PartitionKey=partition-1
```

When we run our application using the **localstack profile** we should see the following logs.

```sh
INFO : Received Kinesis message: KinesisClientRecord(sequenceNumber=49626058596308151669577377721366592164732241681909284866, approximateArrivalTimestamp=2022-01-23T12:24:27.560Z, data=java.nio.HeapByteBufferR[pos=0 lim=11 cap=11], partitionKey=partition-1, encryptionType=NONE, subSequenceNumber=0, explicitHashKey=null, aggregated=false, schema=null).
```

Hooray! We've successfully implemented the Kinesis Client Library, and made it run locally using our own localstack environment 🎉

Want to see the full application? Here is the [Github Repository](https://github.com/VR4J/java-kinesis-client-consumer).
