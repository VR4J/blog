---
title: Jackson builders in Spring Boot Native
date: 2023-02-20 12:00:00
icon: 
image: /images/xxx.jpg
tags: 
    - Spring Native
    - Spring Boot 3
    - GraalVM
    - Jackson
    - Builder
    - Lombok
excerpt: When you love the builder pattern, and compiling to native images as much as me, you need this Jackson configuration in your Spring Boot 3 application.
---

## Introduction

When you've read a few of my earlier posts, you know my secret love for the builder pattern to create immutable objects, the Jackson library for being able use this pattern for deserialization, ánd Lombok for providing the perfect glue between these two in just two annotations.

When you've not read my earlier posts (yes, shame on you) here is the link: [What I love about Lombok and the builder pattern](/2022/01/17/what-i-love-about-lombok-and-the-builder-pattern/)

## Shortcut
Only interested in the code? Look at the final [gist](https://gist.github.com/VR4J/0b30b2d04f95ab37d867353ebcda3e81).

## What's the problem?
When using this in a regular Java Virtual Machine environment, everything is fine, everything works fine because it allows for reflection during runtime.

When using native compilation, which was recently released in Spring Boot 3, you cannot rely on this reflection during runtime and need to configure it upfront during build time.

The Runtime Exception will look something like the following.
```sh
com.fasterxml.jackson.databind.exc.InvalidDefinitionException: Builder class nl.vreijsenj.blog.Movie$Builder does not have build method (name: 'build')
```

## Native Configuration
Spring Boot 3 allows for what's called "native hints" to be declared which are used during build time. These Native Hints allow you to specify classes of which use reflection, or specify resources that would normally be excluded by the native build but are actually needed for a specific use-case.

Spring Boot's way of specifying these native hints is by implementing the [RuntimeHints](https://docs.spring.io/spring-framework/docs/6.0.0/reference/html/core.html#aot-hints) API.

## Identifying Jackson Builders
We first need to be able to identify the builder classes that we need to configure reflection for.

As you know Lombok's `@Jacksonized` annotation will put Jackson's `@JsonPOJOBuilder` annotations on the builder class which we can use to identify our classes. 

Let's see how that would look;

```java
List<TypeReference> classes = getClasses(loader, ROOT_PACKAGE).stream()
    .filter(clazz -> clazz.getAnnotationsByType(JsonPOJOBuilder.class).length > 0)
    .map(TypeReference::of)
    .toList();
```


## Completing the puzzle
When we put the above into our own implementation of a `RuntimeHintsRegistrar` we get the following class.

```java
@Configuration
@ImportRuntimeHints(JacksonHints.class)
public class JacksonHints implements RuntimeHintsRegistrar {

    private static final String ROOT_PACKAGE = "nl.vreijsenj.streaming";
    private static final String PACKAGE_SEPARATOR = ".";
    private static final String FOLDER_SEPARATOR = "/";

    @Override
    public void registerHints(RuntimeHints hints, ClassLoader loader) {
        List<TypeReference> builders = getClasses(loader, ROOT_PACKAGE).stream()
            .filter(clazz -> clazz.getAnnotationsByType(JsonPOJOBuilder.class).length > 0)
            .map(TypeReference::of)
            .toList();

        hints
            .reflection()
            .registerTypes(builders, TypeHint.builtWith(MemberCategory.INVOKE_DECLARED_CONSTRUCTORS, MemberCategory.INVOKE_PUBLIC_METHODS))
    }

    public Set<Class<?>> getClasses(ClassLoader loader, String name) {
        InputStream stream = loader.getResourceAsStream(
                name.replaceAll("[" + PACKAGE_SEPARATOR + "]", FOLDER_SEPARATOR)
        );

        BufferedReader reader = new BufferedReader(new InputStreamReader(stream));
        List<String> lines = reader.lines().toList();

        Stream<Class<?>> current = lines.stream()
            .filter(this::isClassFile)
            .map(line -> getClass(line, name));

        Stream<Class<?>> nested = lines.stream()
            .filter(this::isPackageFolder)
            .map(String::trim)
            .map(child -> setChildPackageName(name, child))
            .map(pName -> getClasses(loader, pName))
            .flatMap(Set::stream);

        return Stream.concat(current, nested).collect(Collectors.toSet());
    }

    @SneakyThrows
    private Class<?> getClass(String className, String packageName) {
        return Class.forName(packageName + PACKAGE_SEPARATOR + className.substring(0, className.lastIndexOf(PACKAGE_SEPARATOR)));
    }

    private boolean isClassFile(String path) {
        return path.endsWith(".class");
    }

    private boolean isPackageFolder(String path) {
        return ! isClassFile(path);
    }

    private String setChildPackageName(String parent, String child) {
        return parent + PACKAGE_SEPARATOR + child;
    }
}
```

## Summary
Well, there you have it, a class ready to be included in any Spring Boot 3 application.
Rather prefer a gist? Way ahead of you [here is the gist](https://gist.github.com/VR4J/0b30b2d04f95ab37d867353ebcda3e81).

That’s all folks! 👋







