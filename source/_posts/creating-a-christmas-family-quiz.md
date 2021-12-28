---
title: Building a multi-device websocket based pubquiz
icon: 🎄
tags:
    - Spring Boot
    - Webflux
    - Java 17
    - Websockets
excerpt: Christmas; what a wonderfull time, wrapping paper in every store, lights and decorations on every corner what's not to like? Well here is the thing, I tend to get bored quite easily, which is why on this years family dinner; I tried something different!
---

Christmas is a time of beatiful lid homes, christmas trees, gifts and... akward family dinners. Unfortunately mine tend to take ages, while sitting and talking about the weather, politics, and other *&lt;sarcasm&gt;* very exciting stuff *&lt;/sarcasm&gt;*. 

## Let's try something different
Well this year I'm going to build an interactive quiz where we'll display questions and riddles on the tv screen while everyone can answer them using their own cellphones.

Just interested in the source code? Check out the [backend repository](https://github.com/VR4J/quizzer-service) and [front-end repository](https://github.com/VR4J/quizzer-front-end) on Github.

## The bigger picture
<img class="right-side" src="/images/the-bigger-picture.jpeg" />

<p style="text-align: justify;">Since we want the whole family to play, we have multiple phones that will be connected to our backend service, while only one tv will be displaying the questions, timers, etc. Meaning we will have two different kind of clients, a <b>player</b> and <b>observer</b>.</p>

<p style="text-align: justify;">Both will be connected to our websocket server which will hold the responsibility to align every client on the state of the quiz.</p>

## Setting up our websocket server
As we said we're going to use a Spring Boot application for our websocket server, so let's set that up.

```groovy
plugins {
	id 'org.springframework.boot' version '2.5.6'
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

// Since we want to use the preview features of Java 17
tasks.withType(JavaCompile) {
	options.compilerArgs += "--enable-preview"
}

dependencies {
	annotationProcessor("org.springframework.boot:spring-boot-configuration-processor")

	implementation("org.springframework.boot:spring-boot-starter-webflux")
	implementation('org.springframework:spring-context')

	implementation('commons-io:commons-io:2.11.0')

	implementation("javax.servlet:javax.servlet-api:4.0.1")

        // Just because we love Lombok <3
	compileOnly 'org.projectlombok:lombok:1.18.20'
	annotationProcessor 'org.projectlombok:lombok:1.18.20'

	testImplementation 'org.springframework.boot:spring-boot-starter-test'
}

test {
	useJUnitPlatform()
}
```

Let's setup the Websocket configuration
```java
@Configuration
@RequiredArgsConstructor
public class WebsocketConfiguration {

    private final WebSocketHandler handler;

    @Bean
    public HandlerMapping webSocketHandlerMapping() {
        Map<String, WebSocketHandler> urlMap = new HashMap<>(){{
            put("/websocket", handler);
        }};

        SimpleUrlHandlerMapping handlerMapping = new SimpleUrlHandlerMapping();
        handlerMapping.setOrder(1);
        handlerMapping.setUrlMap(urlMap);

        return handlerMapping;
    }

    @Bean
    public WebSocketHandlerAdapter handlerAdapter() {
        return new WebSocketHandlerAdapter();
    }
}
```
We can now define our websocket handler, which allow us to start communicating to clients. Since we're using Webflux we can only send data to the client by using a [Reactive Streams Publisher](https://www.reactive-streams.org/reactive-streams-1.0.3-javadoc/org/reactivestreams/Publisher.html) which we'll implement shortly.
```java
@Slf4j
@Component
public class WebsocketHandler implements WebSocketHandler {

    @Override
    public Mono<Void> handle(WebSocketSession session) {
        // TODO
        return Mono.empty()
    }
}
```

## Let's handle player registration
Since uncle Joe is not as familiar with his new iPhone yet, we need to make sure losing connection is not an issue and we can reconnect with a new websocket connection to an existing player. We can achieve this by letting every client / phone send its own unique session id, and store it for future reconnects.

```js
const socket = new WebSocket(`ws://${ip}:8081/websocket?session_id=${session_id}`);
```

Since the backend will now get a unique identifier for every player, we can store and track all players and connect a *Sink* to every client to allow sending messages to specific players.

```java
private final MessagingService service;
private final MessageListener listener;

@Override
public Mono<Void> handle(WebSocketSession session) {
    String sessionId = getSessionId(session.getHandshakeInfo().getUri());

    // Get stream of messages specific for this player / observer
    Flux<WebSocketMessage> messages = service.getMessages(sessionId)
            .map(session::textMessage);

    // Get stream of messages coming from this specific player / observer
    Flux<WebSocketMessage> reading = session.receive()
            .doOnNext(message -> onMessage(message.getPayloadAsText(), sessionId)) ;

    return session.send(messages).and(reading);
}
```
We'll create a *MessagingService* which sole responsibility is to keep track of the session Sinks and expose a method to send a message to a specific session.

```java
@Slf4j
@Component
@RequiredArgsConstructor
public class MessagingService {

    private static final Map<String, Sinks.Many<String>> sinks = new HashMap<>();

    private final ObjectMapper mapper;

    public void onNext(Message next, String session) {
        if(! sinks.containsKey(session)) return;

        try {
            String payload = mapper.writeValueAsString(next);
            sinks.get(session).emitNext(payload, Sinks.EmitFailureHandler.FAIL_FAST);
        } catch (JsonProcessingException e) {
            log.error("Unable to send message {} to session {}", next, session, e);
        }
    }

    public Flux<String> getMessages(String session) {
        sinks.putIfAbsent(session, Sinks.many().multicast().onBackpressureBuffer());
        return sinks.get(session).asFlux();
    }
}
```

## Acting on messages received
Now that we've established a connection to each client, and can individually send messages to them, we can start building our quizzing logic.

Our process is mainly built based on a *"request-reply"* principle, where the following steps are present:
#### Connecting
1. Observer connects
2. Player connects
3. Player sends player name
4. Observer receives player joined (optional)

#### Playing Question
&nbsp;&nbsp; 4. Observer requests next question
&nbsp;&nbsp; 5. Observer receives next question
&nbsp;&nbsp; 6. Player receives multiple choice answers to question

#### Answering
&nbsp;&nbsp; 7. Player sends *picked answer* message
&nbsp;&nbsp; 8. Observer receives player answered 'in x seconds' message

#### Requesting Results
&nbsp;&nbsp; 9. Observer requests question results
&nbsp;&nbsp; 10. Observer receives players result messages
&nbsp;&nbsp; 11. Player receives correct answer feedback

#### Checking Leaderboard
&nbsp;&nbsp; 12. Observer requests leaderboard
&nbsp;&nbsp; 13. Observer receives leaderboard


```java
@Slf4j
@Component
@RequiredArgsConstructor
public class MessageListener {

    private final Observer observer;

    private final Quiz quiz;
    private final Leaderboard leaderboard;

    public void onMessage(Message wsMessage, String sessionId) {
        MessageType type = wsMessage.getType();

        switch (type) {
            case OBSERVER_JOIN -> {
                observer.setSessionId(sessionId);
            }
            case PLAYER_JOIN -> {
                PlayerJoinMessage message = (PlayerJoinMessage) wsMessage;

                Player player = Player.builder()
                        .id(sessionId)
                        .name(message.getName())
                        .score(0)
                        .build();

                quiz.register(player, sessionId);

                // Notify the observer of a player join.
                observer.send(message); 
            }
            case SHOW_QUESTION -> {
                // Observer requesting next question
                quiz.next();
            }
            case ANSWER -> {
                // Player answering to question
                AnswerMessage message = (AnswerMessage) wsMessage;
                quiz.answer(sessionId, message.getAnswer(), message.getScore());
            }
            case TIMEOUT -> {
                quiz.showQuestionResult();
            }
            case SHOW_LEADERBOARD -> {
                // Observer requesting the leaderboard
                leaderboard.show();
            }
        }
    }
}

```

There you have it, the process of my own anti-boredom family pub quiz.

As you can see I've left the actual quiz, and leaderboard functionality out of this blogpost, but the full implementation details are available in the [Github Repository](https://github.com/VR4J/quizzer-service).

### Demo

Curious to how it all works together? Wait no longer, here is a short dutch demo!

<video src="/videos/quizzer-demo.mov" width="100%" controls />