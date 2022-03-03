---
title: (De)-serializing Polymorphic objects with Jackson
date: 2022-03-03 11:30:00
icon: 🐕
image: /images/de-serializing-polymorphic-objects-with-jackson.png
tags:
    - Java 17
    - Spring Boot
    - Jackson
    - Polymorphism
    - Descriminator
excerpt: Polymorphism is all cool and hipster, but how do you deserialize such structure coming from the outside world? Well, I so happen to have written this blogpost about it so click along!
---

We've all heard about the `class Dog extends Animal` concept at some point of our wonderful programming lives, but have you ever heard about how to maintain such polymorphic structure when (de-)serializing the Dog class?

## What is Polymorphism?

### Theory
With subjects like this there is already a lot of information available on the world-wide web, and explaining polymorphism couldn't be done better than how Torben Janssen described it on [Stackify.com](https://stackify.com/oop-concept-polymorphism/)

> Polymorphism describes situations in which something occurs in several different forms. In computer science, it describes the concept that you can access objects of different types through the same interface.

That last part is exactly what we want to achieve: deserializing a payload by specifying the interface without knowing the exact implementation that will be provided.

### Example
Since you've heard the animal example a hundred times already, we'll use a more real-life example and consider the following list of **Messages**.

```json
[
    {
        "type": "TEXT",
        "text": "Hello World!"
    }, {
        "type": "CONTACT",
        "contact": {
            "forename": "Joery",
            "surname": "Vreijsen",
            "phone": "+123456790"
        }
    }, {
        "type": "LOCATION",
        "location": {
            "latitude": 50.894941,
            "longitude": 4.341547
        }
    }
]
```

## Implementation

With that we can start defining that interface from which we can access all different types of messages.

```java
interface Message {
    MessageType getType();
}
```

We can use the interface above that defines the message type, or we can define an abstract class like below to prevent duplicating constructor logic in every implementation.

```java
abstract class Message {
    protected MessageType type;

    public Message(MessageType type) {
        this.type = type;
    }

    public MessageType getType() {
        return this.type;
    }
}
```

By extending the abstract class we can now define our specific type of messages, starting with the obvious **TextMessage**.

```java
class TextMessage extends Message {

    private final String text;

    public TextMessage(@JsonProperty("text") String text) {
        super(MessageType.TEXT);

        this.text = text;
    }

    public String getText() {
        return this.text;
    }
}
```

Next to text messages we can also share locations, or contacts by sending **LocationMessages** or **ContactMessages**.

```java
class LocationMessage extends Message {

    private final Location location;

    public LocationMessage(@JsonProperty("location") Location location) {
        super(MessageType.LOCATION);

        this.location = location;
    }

    public Location getLocation() {
        return this.location;
    }
}
```

```java
class ContactMessage extends Message {

    private final Contact contact;

    public ContactMessage(@JsonProperty("contact") Contact contact) {
        super(MessageType.CONTACT);

        this.contact = contact;
    }

    public Contact getContact() {
        return this.contact;
    }
}
```

Alright, we have got all our models setup but we can't quite deserialize the json from above yet because Jackson doesn't know how to map a type to a specific class.
Here's the trick; we can add the Jackson `@JsonTypeInfo` annotation to specify that specific mapping on our Message interface or abstract class.

```java
@JsonTypeInfo(
        use = JsonTypeInfo.Id.NAME,
        property = "type"
)
@JsonSubTypes({
        @JsonSubTypes.Type(value = TextMessage.class, name = "TEXT"),
        @JsonSubTypes.Type(value = LocationMessage.class, name = "LOCATION"),
        @JsonSubTypes.Type(value = ContactMessage.class, name = "CONTACT")
})
public abstract class Message {
    protected MessageType type;

    public Message(MessageType type) {
        this.type = type;
    }

    public MessageType getType() {
        return this.type;
    }
}
```

Let's step back a bit shall we? We added the `@JsonTypeInfo` specifying which property (type) should be used to descriminate the message.
We then add the `@JsonSubTypes` annotation to specify the mapping of the property to the respective implementation.

As simple as that, we can now deserialize the json using Jackson's ObjectMapper.

```java
class Main {

    // Java 15+ Text Blocks 🎉
    private static final String json =
            """
                [
                    {
                        "type": "TEXT",
                        "text": "Hello World!"
                    }, {
                        "type": "CONTACT",
                        "contact": {
                            "forename": "Joery",
                            "surname": "Vreijsen",
                            "phone": "+123456790"
                        }
                    }, {
                        "type": "LOCATION",
                        "location": {
                            "latitude": 50.894941,
                            "longitude": 4.341547
                        }
                    }
                ]
            """;

    public static void main(String[] args) throws JsonProcessingException {
        List<Message> messages = new ObjectMapper().readValue(json, new TypeReference<>() { });
        System.out.println(messages);
    }
}
```

## Result

When we run this we see the following output confirming that we deserialized the json to our specific message classes.

```txt
[nl.vreijsenj.messaging.TextMessage@64485a47, nl.vreijsenj.messaging.ContactMessage@25bbf683, nl.vreijsenj.messaging.LocationMessage@6ec8211c]
```

Hooray! We could still extend our message class with properties like `recipient`, `sender` or add abstract methods for the specific classes to implement.