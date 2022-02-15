---
title: A story on how to use delay queues
date: 2022-02-15 11:30:00
icon: ⏱️
image: /images/creating-time-based-escalations-using-an-sqs-queue-social.jpg
tags:
    - Java 17
    - Spring Boot
    - AWS
    - SQS
    - Delay Queue
excerpt: Ever heard about delay queues, and how they could be used? Join me in this real-life use-case where we will utilise delay queues to expose critical issues.
---

Are you looking for information on delay queues and how they should be implemented using AWS SQS? Funny, that's exactly what i'm going to talk about.

As we're on the verge of a new Formula 1 season, lets use a real-life racing use-case. Nowadays Formula 1 cars are full of sensors to monitor everything you can imagine on a car, from tire and engine temperatures to [slow punctures](https://www.tyrepros.co.uk/blog/how-to-identify-a-slow-puncture-and-what-you-should-do-about-it).

Not such a Formula 1 fan? Skip to [the theory](#The-Theory), or check out the [Github Repository](https://github.com/VR4J/sqs-delayed-escalation-service) for the final code.

## The Goal
During a race, engine temperatures can rise for a number of reasons without it immediately meaning there is an issue. For example, our car is following another car closely and is therefore picking up hot air from the car in front, making our engine temperatures run high. Generally this isn't a problem, as running closely behind another car is needed to make an overtake, but if the driver fails to make a successfull overtake and keeps driving within this hot air for a prolonged period of time, the risk of engine damage becomes bigger and bigger.

Since we have excellent sensors on our car, we receive real-time engine temperature updates and can notify the pitwall whenever the engine temperature has been too high for too long.

## The Theory
Let's get theoretical shall we?

<img src="/images/delay-queues-explained.svg" />

Based on the schematic above we can list the responsibilities of our Spring Boot service:
1. Receive temperature changes, and keep the engine state updated.
3. Whenever the temperature has crossed our threshold, send a **delayed** temperature check message to the queue.
4. Receive the delayed message and check whether the engine temperature is still above our threshold and if so, raise the severity.
5. Send notifications on severity changes.

## The Execution
On to the implementation we go and first things first, we setup the Gradle project using the following **build.gradle** file.

```groovy
plugins {
	id 'org.springframework.boot' version '2.6.3'
	id 'io.spring.dependency-management' version '1.0.11.RELEASE'
	id 'java'
}

group = 'com.vreijsenj'
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
	implementation("org.springframework:spring-context")

	implementation("javax.servlet:javax.servlet-api:4.0.1")

	// SQS Client Library
	implementation("software.amazon.awssdk:sqs:2.17.123")
	implementation("com.amazonaws:amazon-sqs-java-messaging-lib:1.0.8")


	compileOnly("org.projectlombok:lombok:1.18.20")
	annotationProcessor("org.projectlombok:lombok:1.18.20")
	testImplementation("org.springframework.boot:spring-boot-starter-test")
}

tasks.named('test') {
	useJUnitPlatform()
}
```

Now that we got that out of the way, wouldn't it be nice to be able to use JMS with SQS?

Well, we weren't the first ones to think of that, if we look at the `amazon-sqs-java-messaging-lib` library, we can see that there is an `SQSConnectionFactory` that implements the JMS `ConnectionFactory` interface. 

Alright, so once we create a bean for it we should be able to create a connection, add a message listener and get on our way!

```java
@Configuration
@Profile("!localstack")
public class ClientConfig {

    @Bean
    public AmazonSQS sqs() {
        return AmazonSQSClientBuilder.defaultClient();
    }

    @Bean
    public SQSConnectionFactory sqsConnectionFactory(AmazonSQS sqs) {
        ProviderConfiguration configuration = new ProviderConfiguration()
                .withNumberOfMessagesToPrefetch(10);

        return new SQSConnectionFactory(configuration, sqs);
    }
}
```

Notice that we put the `@Profile("!localstack")` on it, as in the [Github Repository](https://github.com/VR4J/sqs-delayed-escalation-service) there is also a `@Profile("localstack")` configuration which allows us to connect to a localstack environment to test the integration on our own computer.

Right, so we got the **SQSConnectionFactory** bean, lets create a connection and register a message listener!

```java
@Component
@RequiredArgsConstructor
public class SqsConfig {

    private final SQSConnectionFactory sqsConnectionFactory;
    private final SQSConnectionProperties properties;
    private final MessageListener sqsMessageListener;

    @PostConstruct
    public void registerQueueListener() throws JMSException {
        SQSConnection connection = sqsConnectionFactory.createConnection();
        Session session = connection.createSession(false, Session.AUTO_ACKNOWLEDGE);
        MessageConsumer consumer = session.createConsumer(session.createQueue(properties.getName()));

        consumer.setMessageListener(sqsMessageListener);
        connection.start();
    }
}
```

Let's make a quick pitstop; As you can see in addition to the **SQSConnectionFactory**, we also autowired a **ConfigurationProperties** bean to be able to specify our target sqs queue in the `application.yaml`, but also a JMS MessageListener bean to be registered when creating the connection.

In the **SQSMessageListener** we will make a distinction between the two types of messages that we can receive:
1. [Temperature Change Message](https://github.com/VR4J/sqs-delayed-escalation-service/blob/main/src/main/java/com/vreijsenj/escalation/temperature/TemperatureChangeMessage.java): Comes directly from our engine, and contains the current temperature.
2. [Temperature Check Message](https://github.com/VR4J/sqs-delayed-escalation-service/blob/main/src/main/java/com/vreijsenj/escalation/temperature/TemperatureCheckMessage.java): Scheduled by ourselves to check whether the temperature is still above a certain threshold.

So before we can start writing logic on each one of them we have to identify which message was received.

```java
@Slf4j
@Component
@RequiredArgsConstructor
public class SqsMessageListener implements MessageListener {

    public static final String CHECK_MESSAGE = "CHECK";
    public static final String CHANGE_MESSAGE = "CHANGE";

    private final ObjectMapper mapper;
    private final TemperatureCheckListener temperatureCheckListener;
    private final TemperatureChangeListener temperatureChangeListener;

    @Override
    public void onMessage(Message message) {
        if (message instanceof SQSTextMessage sqsTextMessage) {
            try {
                onMessage(sqsTextMessage);
            } catch (JMSException | JsonProcessingException e) {
                log.error("Something went wrong retrieving the payload of the SQSTextMessage.", e);
            }
        }
    }

    private void onMessage(SQSTextMessage sqsTextMessage) throws JMSException, JsonProcessingException {
        TemperatureMessage message = mapper.readValue(sqsTextMessage.getText(), TemperatureMessage.class);

        switch (message) {
            case TemperatureCheckMessage check -> {
                temperatureCheckListener.onTemperatureCheck(check);
            }
            case TemperatureChangeMessage change -> {
                temperatureChangeListener.onTemperatureChange(change);
            }
            default -> { }
        }
    }
}
```

*Bonus: Did you notice how we used the Java 17 pattern matching in our switch statement above?*

Alright, so we can now identify each message that comes in and call its respective listener.

Let's start with the **TemperatureChangeMessageListener** who has the responsibility of keeping the EngineState up to date and trigger a TemperatureCheckMessage whenever the threshold was exceeded.

```java
@Slf4j
@Component
@RequiredArgsConstructor
public class TemperatureChangeListener {

    private final SQSConnectionProperties sqsProperties;
    private final EscalationConfigurationProperties escalation;

    private final TemperatureCheckScheduler scheduler;
    private final EngineState engine;

    public void onTemperatureChange(TemperatureChangeMessage message) throws JsonProcessingException {
        log.info("Temperature Change: {}", message.getTemperature());

        Double temperature = message.getTemperature();

        engine.setTemperature(temperature);

        if(temperature > escalation.getThreshold().getTemperature()) {
            scheduler.schedule(sqsProperties.getUrl(), escalation.getDelay(), engine.getSeverity());
        }
    }
}
```

Once again we autowire our **SQSConnectionProperties** in case we need to send out a temperature check message, and we autowire the **EscalationConfigurationProperties** which define the temperature threshold and the delay in which the engine temperature should have returned to normal.

We update the **EngineState** and afterwards we check whether the temperature has exceeded the threshold, if so we let the **TemperatureCheckScheduler** schedule a message to check the temperature for when the delay has passed.

```java
@Component
@RequiredArgsConstructor
public class TemperatureCheckScheduler {

    private final SqsClient sqs;
    private final ObjectMapper mapper;

    public void schedule(String queue, Integer delay, EngineState.Severity severity) throws JsonProcessingException {
        String body = getMessagePayload(severity);

        SendMessageRequest request = SendMessageRequest.builder()
                .queueUrl(queue)
                .delaySeconds(delay)
                .messageBody(body)
                .build();

        sqs.sendMessage(request);
    }

    private String getMessagePayload(EngineState.Severity severity) throws JsonProcessingException {
        return mapper.writeValueAsString(
                new TemperatureCheckMessage(CHANGE_MESSAGE, severity)
        );
    }
}
```

In the **TemperatureCheckMessage** we put the severity that the engine currently has, so that when the delay has expired and the temperature is still too high, we know to what severity the engine state should be raised.

*Best practice:* When running this in production, it might be best to put these messages on a seperate queue to avoid any additional delays caused by other messages in the queue.

Note here the *delaySeconds* in the request that prevents the message from being picked up before the delay has expired.

Alright so we have completed the logic for our **TemperatureChangeMessage**, now lets look at what we should do when we receive our delayed **TemperatureCheckMessage**.

```java
@Slf4j
@Component
@RequiredArgsConstructor
public class TemperatureCheckListener {

    private final EngineState engine;
    private final EscalationConfigurationProperties escalation;

    public void onTemperatureCheck(TemperatureCheckMessage message) {
        if(engine.getTemperature() > escalation.getThreshold().getTemperature()) {
            EngineState.Severity previousSeverity = message.getSeverity();
            EngineState.Severity nextSeverity = previousSeverity == EngineState.Severity.CLEAR
                ? EngineState.Severity.WARNING
                : EngineState.Severity.CRITICAL;

            if(engine.getSeverity() == nextSeverity) return;

            log.info("Sending notification: Engine temperature has been above configured threshold of '{}' degrees celsius.", escalation.getThreshold().getTemperature());
            log.info("Engine severity raised to {}", nextSeverity);
            engine.setSeverity(nextSeverity);
        }
    }
}
```

First thing first, we check whether the engine temperature is still above the configured threshold, otherwise we don't care. We could extend the logic to de-escalate when the engine has gone below the threshold, but lets leave that for now.

So the temperature is still too high and based on the previous severity in the message, we can determine what the next severity should be. As we can have multiple temperature readings within the configured delay we only update the state, and send notifications, if the engine severity has changed.

That's it, we should now be able to escalate temperature readings to our pitwall based on our configured temperature threshold!

## The Test

When we create the following configuration in our application.yaml:
```yaml
sqs:
  connection:
    name: engine-temperatures
    url: http://localhost:4566/000000000000/engine-temperatures

escalation:
  delay: 60
  threshold:
    temperature: 50.0
```

And we create the following queue using the awslocal cli:
```sh
$ awslocal sqs create-queue --queue-name engine-temperatures
{
    "QueueUrl": "http://localhost:4566/000000000000/engine-temperatures"
}
```

And we send in the following engine temperatures using the awslocal cli:
```sh
awslocal sqs send-message --queue-url http://localhost:4566/000000000000/engine-temperatures --message-body '{ "type": "CHANGE", "temperature": 48.0 }'; \
sleep 10; \
awslocal sqs send-message --queue-url http://localhost:4566/000000000000/engine-temperatures --message-body '{ "type": "CHANGE", "temperature": 53.0 }'; \
sleep 10; \
awslocal sqs send-message --queue-url http://localhost:4566/000000000000/engine-temperatures --message-body '{ "type": "CHANGE", "temperature": 54.0 }'; \
sleep 60; \
awslocal sqs send-message --queue-url http://localhost:4566/000000000000/engine-temperatures --message-body '{ "type": "CHANGE", "temperature": 58.0 }'
```

Then we should see the following log output:
```sh
2022-02-07 15:00:46.166  INFO : Temperature Change: 48.0
2022-02-07 15:00:56.777  INFO : Temperature Change: 53.0
2022-02-07 15:01:07.677  INFO : Temperature Change: 54.0
2022-02-07 15:01:56.950  INFO : Sending notification: Engine temperature has been above configured threshold of '50.0' degrees celsius.
2022-02-07 15:01:56.950  INFO : Engine severity raised to WARNING
2022-02-07 15:02:08.359  INFO : Temperature Change: 58.0
2022-02-07 15:03:08.414  INFO : Sending notification: Engine temperature has been above configured threshold of '50.0' degrees celsius.
2022-02-07 15:03:08.414  INFO : Engine severity raised to CRITICAL
```

Call me a Geek, but if that's not cool, i don't know what is.

Hope you enjoyed the long post, as always if you're interested in the whole code base; the [Github Repository](https://github.com/VR4J/sqs-delayed-escalation-service) is just one click away!




