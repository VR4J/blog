---
title: 'Cloud Native Development: Are you prepared?'
date: 2022-08-02 11:30:00
icon: 🧙‍♂️
image: /images/cloud-native-development-are-you-prepared-social.png
tags:
    - Cloud Native
    - GraalVM
    - Spring Boot
    - Spring Native
    - Quarkus
    - Java 17
excerpt: Whether you like it or not, cloud native development is rising in popularity. Question is, are you prepared to adapt to it?
---

Let's first establish a common understanding on the term 'Cloud Native', as it tends to differ from person to person... or even from     day to day.

> Cloud-native is a modern approach to building and running software applications that exploits the flexibility, scalability, and resilience of cloud computing.

Alright so in other words, to become cloud-native, to fully exploit the felxibility, scalability, and resilience of cloud computing, we need to change our approach in which we are building and running our software.

### What's so different?
When you develop your typical Java application, you know it's going to run in our beloved Java Virtual Machine (JVM). Our magic box for translating Java bytecode to machine specific code, and managing our memory consumption. Apart from the that, it has some other smart tricks up its sleeves to allow our code to run as fast and efficient as possible, think about [warmup](https://devcenter.heroku.com/articles/warming-up-a-java-process), [garabage collection](https://www.eginnovations.com/blog/what-is-garbage-collection-java), etc.

On the downside, these smart tricks don't come for free, we all know the JVM is quite slow in its start up and uses additional memory and cpu to perform for example the garbage collection tasks. 

When doing Cloud Native Development in Java we have to compile our application directly into a binary file ready to be executed on the targets machines bytecode, no Java Virtual Machine involved.

### What's the benefit?
When having a binary specificly build for your targets host, it's lightning fast in startup allowing to fully benefit from the scalability cloud computing offers. Apart from that there is no need for a magic box like the JVM to translate Java bytecode to machine code on the fly, we don't need any smart tricks to keep part of our machine bytecode in a cache for faster executions (warmup). 

Down the line we could say that, by erraticating the need for the JVM, we significantly decrease both the startup time as well as the overall memory footprint of the application.

### How does it work?
How do we build this lightning fast binary that you so highly speak of, you ask? Well, let me introduce you to a buddy of mine, [GraalVM](https://www.graalvm.org/).

GraalVM is a Java VM that allows for *ahead-of-time compilation*, which is best explained by this little snippet from [wikipedia](https://en.wikipedia.org/wiki/Ahead-of-time_compilation).
> The act of compiling a higher-level programming language such as C or C++, or an intermediate representation such as Java bytecode, into a native (system-dependent) machine code so that the resulting binary file can execute natively.

So in short, GraalVM allows us to turn our Java application into a single binary file that can be executed natively.

### What's the catch?
You'll probably be wondering well.. that sounds like sunshine and rainbows but what's the catch here? Well as good as a buddy GraalVM is, it tends to be a bit difficult to handle. With the use of *ahead-of-time compilation* there are some restrictions, like the use of reflection in Java. Reflection allows the Java application to "introspect" upon itself during runtime, which simply will not be possible if you already compiled everything upfront into a binary file.

As reflection is deeply nested into the core of most Java applications; think about serialization, or dependency injection, it can be quite difficult to eliminate it completely. 

On the upside, we already see movement in the opensource world where projects are starting to move away from using reflection, or finding workarounds specifically for these kind of purposes. 

[Quarkus](https://quarkus.io/) for example is one of, if not the biggest, driver within the Java world that embraces the "container first" principle, tailoring applications for GraalVM. Spring Boot is slowly catching on as well with its own project called [Spring Native](https://github.com/spring-projects-experimental/spring-native).

With the arrival of *ahead-of-time compilation* we do have to admit that compiling an application with Java takes significantly longer. Building the full native image can range from 5 minutes up to 25 minutes, depending on how many dependencies are involved, and how big of an application you are building.

While this ofcourse does not take away the fact that you can still run your good old jar file for local testing, it does not fully represent how the binary executable will run or behave.

### What's your verdict?
While I strongly believe that cloud native development will be a thing in the future, there is also alot of work that still needs to be done to make it more developer friendly. Currently it requires alot of knowledge on the internals of GraalVM, to fully be able to use it in your own project, and requires lots of fiddling with different options to get the result you want.

I'm sure projects like Quarkus and Spring Native will soon make this a thing of the past, so for now **consider cloud-native development, but be sure to know what you're in for!**
