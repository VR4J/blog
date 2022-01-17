---
title: What I love about Lombok and the builder pattern
date: 2022-01-17 09:15:00
icon: 🏗️
image: /images/what-i-love-about-lombok-and-the-builder-pattern.jpg
tags:  
    - Lombok
    - Jackson
    - Builders
excerpt: We all know I'm a big fan of Lombok, but why? Well here are a few reasons that will make sure you start using it today!
---

First things first, let's quickly look at what Lombok exactly is, i'll do one better; let's look at how they describe it themselves at https://projectlombok.org:
> Project Lombok is a java library that automatically plugs into your editor and build tools, spicing up your java.
> Never write another getter or equals method again, with one annotation your class has a fully featured builder, Automate your logging variables, and much more.

Exactly, the **and much more** is key in this little paragraph.

## The Builder Pattern
As you probably already know, the builder pattern is one of the design patterns used in object-oriented programming. It allows for a simple, easy to read and immutable way of contructing objects. Your object will have no setters, nor public constructors, rather a static builder class that allows for the creation of the object. By calling the builder class' methods you can decide which properties you want to provide, and just leave out the ones you don't care about.

Well that's all nice and theoretic but how does it look?

```java
public class BlogPost {

    private String title;
    private String author;
    private String category;
    private Instant created;

    BlogPost(String title, String author, String category, Instant created) {
        this.title = title;
        this.author = author;
        this.category = category;
        this.created = created;
    }

    public static BlogPostBuilder builder() {
        return new BlogPostBuilder();
    }

    public static class BlogPostBuilder {
        private String title;
        private String author;
        private String category;
        private Instant created;

        private BlogPostBuilder() { }

        public BlogPostBuilder title(String title) {
            this.title = title;
            return this;
        }

        public BlogPostBuilder author(String author) {
            this.author = author;
            return this;
        }

        public BlogPostBuilder category(String category) {
            this.category = category;
            return this;
        }

        public BlogPostBuilder created(Instant created) {
            this.created = created;
            return this;
        }

        public BlogPost build() {
            return new BlogPost(title, author, category, published);
        }
    }
}
```

We can now leverage our BlogPostBuilder to instantiate a BlogPost.

```java
BlogPost.builder()
    .title("Very awesome blog post")
    .author("Joery Vreijsen")
    .category("Java")
    .created(Instant.now())
    .build()
```

So hold on, where does Lombok come in? Well wouldn't it be nice to get rid of all the boilerplate code in the BlogPost class, not having to create a builder method for every property you add? Let me introduce you to Lombok's `@Builder` annotation.

```java
@Builder
public class BlogPost {

    private String title;
    private String author;
    private String category;
    private Instant created;
}
```

Look at that, such a pretty class and since Lombok acts during the compilation process there is no runtime magic going on!

## Builder Defaults

Another great feature of Lombok is the `@Builder.Default` annotation which allows you to specify a default value incase the caller didn't provide a value when constructing the class. For example, lets say we want to default our `created` property to *now* when left out.

```java
@Builder
public class BlogPost {

    private String title;
    private String author;
    private String category;

    @Builder.Default
    private Instant created = Instant.now();
}
```

With this annotation it creates an `if()` statement in the builders `build()` method, which checks whether the property was set, and if not set the default value.

## Deserialisation using a Builder

Usually deserialisation is done by ether leveraging reflection (meh), using the constructors, or calling the setters. With the builder pattern the constructors and setters will not be there.. so good luck with that.

Fortunately Jackson understand the builder pattern and we can annotate our class so Jackson knows its way around it.

```java
@Builder
@JsonDeserialize(builder = BlogPost.BlogPostBuilder.class)
public class BlogPost {

    private String title;
    private String author;
    private String category;

    @Builder.Default
    private Instant created = Instant.now();
    
    @JsonPOJOBuilder(withPrefix = "")
    static class BlogPostBuilder { }
}
```

Alright so let's deep-dive a bit on this, as a lot is going on. First we added the `@JsonDeserialize()` annotation to let Jackson know where it can find our builder class, but unfortunately Jackson isn't aligned with how Lombok thinks builder classes should be defined.

Jackson (by default) assumes the builder class methods are prefixed with `with`. So it would think our builder looks something like this.

```java
BlogPost.builder()
    .withTitle("Very awesome blog post")
    .withAuthor("Joery Vreijsen")
    .withCategory("Java")
    .withCreated(Instant.now())
    .build()
```

That's not the standard Lombok uses, so we need to annotate our builder to instruct Jackson to not assume any prefix. *Fun fact:* with Lombok you can still specify your own builder class, and it will just append it's generated code to it. 

Alright so far, we only added Jackson annotations, so it's not really Lombok that did us any favors here. Since Lombok targets to eliminate as much boilerplating as possible there is another annotation that we can use: `@Jacksonized`. With this annotation Lombok will basically add the Jackson code that we previously created ourselves.. nice right?

```java
@Builder
@Jacksonized
public class BlogPost {

    private String title;
    private String author;
    private String category;

    @Builder.Default
    private Instant created = Instant.now();
}
```

As with everything that is free, there is a disclaimer: The `@Jacksonized` annotation was added as an experimental feature in Lombok 1.18.14 which means that it could be removed in the future.

## Using collections in a Builder

What if we have collections in our class? Like what if our BlogPost could have a Set of categories instead of just one. We can just change the type and Lombok will automatically provide us a builder method where we can pass in a Set of categories, but what if we would want to add them one by one?

That's where the `@Singular` annotation comes it; it allows you to provide just one category which the builder will then add to the Set of categories.

```java
BlogPost.builder()
    .title("Very awesome blog post")
    .author("Joery Vreijsen")
    .category("Java")
    .category("Lombok")
    .created(Instant.now())
    .build()
```

However there is one catch, ones you annotate the collection with `@Singular` the implementation of the `.categories()` method on the builder changes.
Instead of overwriting the whole collection, it will now use the `.add()` method of the collection, which can cause troubles if you first set an immutable collection and then try to add more items to it.

## Changing an object created with a builder class

As we said at the start of this blogpost, the Builder pattern allows us to create immutable objects, which means we can't change them later on. However what we can do is copy it, adjust it, and create a new object of it specifying the `@Builder(toBuilder = true)` property.

```java
BlogPost myFirstBlogPost = BlogPost.builder()
    .title("Very awesome blog post")
    .author("Joery Vreijsen")
    .category("Java")
    .created(Instant.now())
    .build()

BlogPost mySecondBlogPost = myFirstBlogPost.toBuilder()
    .title("Another very awesome blog post")
    .clearCategories()
    .category("Lombok")
    .build()
```

Our second BlogPost now contains everything from our first BlogPost except for the title and categories. Note we use the `clearCategories()` method created by the `@Singular` annotations as the `category()` and `categories()` methods changed implementation to only add to the Set of categories.

That's it! Hope you enjoyed it, and see you next time.