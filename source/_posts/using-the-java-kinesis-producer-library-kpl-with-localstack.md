---
title: Using the Java Kinesis Producer Library (KPL) with LocalStack
date: 2022-11-18 11:30:00
icon: 💨
image: /images/using-the-java-kinesis-producer-library-kpl-with-localstack-social.png
tags:
    - Java 17
    - AWS
    - Kinesis
    - LocalStack
    - Spring Boot
excerpt: The Kinesis Producer Library offers us a great abstraction with a lot of additional configuration options, but setting it up with LocalStack can be tricky. Let's look at how to do it right.
---

Yes! LocalStack, as you know I'm a **huge** fan; being able to run AWS Cloud Services locally is a great way to help test systems that heavily rely on them. Not a fan yet? No worries there's still time, have a look at their website: https://localstack.cloud/.

Only interested in the code? Complete working example is available on [Github](https://github.com/VR4J/kinesis-producer-library-localstack)

## Introduction
Part of the reason Kinesis is so great, is that it is very accessible, either by a few clicks in the AWS Console, or the use of the AWS SDK. The latter however is sometimes a bit of a handful to get setup, but once done the many abstraction layers like the Kinesis Producer Library will give you a lot of 'free' configuration options, and default implementations. We can think of sending messages in batches based on the batch size, or the batch count, preventing our system to send enormous amounts of data over the line in one request.

The Kinesis Producer Library will not work out of the box with LocalStack, but luckily we can fix this with just a few subtle changes.

Ready? Let's dive into it!

## Running LocalStack
LocalStack offers us a Docker image with which we can simply run the necessary AWS services locally.
```sh
docker run --rm -it -e SERVICES=kinesis,cloudwatch,dynamodb -p 4566:4566 -p 4510-4559:4510-4559 localstack/localstack
```

More info about this command can be found on the documentation page: 
https://docs.localstack.cloud/get-started/#docker

When our container is up and running we can use the `awslocal` cli to create a Kinesis Stream.

```sh
awslocal kinesis create-stream --stream-name some-data-stream --shard-count 2
```

## Using the Kinesis Producer Library (KPL)
When we add the Kinesis Producer Library in our Spring Boot application, we need to override a few properties to make sure it connects to our local container.

We can create the following configuration class.

```java
@Profile("localstack")
@Configuration
@RequiredArgsConstructor
public class LocalStackClientConfiguration {
    
    @Bean
    public KinesisProducer kinesisProducer() {
        KinesisProducerConfiguration configuration = new KinesisProducerConfiguration()
            .setKinesisEndpoint("localhost")
            .setKinesisPort(4566)
            .setCloudwatchEndpoint("localhost")
            .setCloudwatchPort(4566)
            .setRegion("us-east-1")

        return new KinesisProducer(configuration);
    }
}
```

Let's explain.

We add the `@Profile("localstack")` annotation to make sure we only connect to our local container when we run our application with the `localstack` profile.

We override the endpoint and ports to match the ones exposed by our LocalStack container. 

Since LocalStack by default registers all resources in the **us-east-1** region, we override it to make sure our stream can be found.

Sounds good? Let's try and send a message to our stream.

```java
@Component
@RequiredArgsConstructor
public class KinesisSender {

    private final KinesisProducer producer;

    @PostConstruct
    public void run() {
        ByteBuffer payload = ByteBuffer.wrap("{ 'data': 'Something important.' }".getBytes(StandardCharsets.UTF_8));
        producer.addUserRecord("some-data-stream", "partitionKey", payload).get();
    }
}
```

Oh oh..

```sh
libc++abi: terminating with uncaught exception of type boost::wrapexcept<boost::exception_detail::error_info_injector<boost::log::v2s_mt_posix::system_error> >: Failed to set TLS value: Invalid argument [system:22]

java.lang.RuntimeException: Child process exited with code 134
    at com.amazonaws.services.kinesis.producer.Daemon.fatalError(Daemon.java:532) ~[amazon-kinesis-producer-0.14.13.jar:na]
    at com.amazonaws.services.kinesis.producer.Daemon.fatalError(Daemon.java:508) ~[amazon-kinesis-producer-0.14.13.jar:na]
    at com.amazonaws.services.kinesis.producer.Daemon.startChildProcess(Daemon.java:486) ~[amazon-kinesis-producer-0.14.13.jar:na]
    at com.amazonaws.services.kinesis.producer.Daemon.access$100(Daemon.java:61) ~[amazon-kinesis-producer-0.14.13.jar:na]
    at com.amazonaws.services.kinesis.producer.Daemon$1.run(Daemon.java:130) ~[amazon-kinesis-producer-0.14.13.jar:na]
    at java.base/java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1136) ~[na:na]
    at java.base/java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:635) ~[na:na]
    at java.base/java.lang.Thread.run(Thread.java:833) ~[na:na]
```

And this is suppose to help us how? 

Well, what they don't tell you, is that **the Kinesis Producer Library (KPL) only works over https**. Which means that it cannot connect to our local container on the plain http port (:4566).

Luckily LocalStack also exposes our AWS services on an https port (:4567), but we still need to re-run our docker container also exposing that port to the host.

```sh
docker run --rm -it -e SERVICES=kinesis,cloudwatch,dynamodb -p 4566:4566 -p 4567:4567 -p 4510-4559:4510-4559 localstack/localstack
```

Alright, so with that done we can change our configuration class.

```java
@Profile("localstack")
@Configuration
@RequiredArgsConstructor
public class LocalStackClientConfiguration {
    
    @Bean
    public KinesisProducer kinesisProducer() {
        KinesisProducerConfiguration configuration = new KinesisProducerConfiguration()
            .setKinesisEndpoint("localhost")
            .setKinesisPort(4567)
            .setCloudwatchEndpoint("localhost")
            .setCloudwatchPort(4567)
            .setVerifyCertificate(false)
            .setRegion("us-east-1");

        return new KinesisProducer(configuration);
    }
}
```

Notice that we also have to set the `setVerifyCertificate()` to `false` since we are connecting locally, and the certificate will not be valid. 

When we try and run our program again, we see no errors and all our messages are being sent! 🎉

That's all folks! 👋 

As always you can find the complete source on [Github](https://github.com/VR4J/kinesis-producer-library-localstack).

## Credits
Special thanks to **Mees van Straten** for pointing out the difficulty of connecting the Kinesis Producer Library to LocalStack.