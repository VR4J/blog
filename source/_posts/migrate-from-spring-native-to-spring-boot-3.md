---
title: Migrating from Spring Native to Spring Boot 3
date: 2022-10-24 15:00:00
image: /images/migrating-spring-native-to-spring-boot-3.jpg
icon: 🍃
tags: 
    - Spring Native
    - Spring Boot 3
    - GraalVM
excerpt: Spring Boot 3 will support native compilation out of the box, we look at how to migrate our project from the Spring Native library.
---

## History Lesson
It was the year 2021.. wait yes that was just last year.. when Spring announced it would be starting to support compilation to native images ([proof](https://spring.io/blog/2021/03/11/announcing-spring-native-beta)). Spring launched the `spring-native` project in beta-support for everyone that wanted to have a lightning fast spring-boot application compiled with GraalVM that could achieve startup times sub 100 ms!

We are now reaching a new milestone in this journey as Spring announced it will move the support for compilation to native images to its General Availability with Spring Framework 6 and Spring Boot 3 ([more proof](https://spring.io/blog/2022/09/26/native-support-in-spring-boot-3-0-0-m5)).  

You as hyped as I am? You need a second to catch your breath? Good, because I couldn't wait either to migrate from the spring-native beta to Spring Boot 3. Since migrating is never an easy task, I've listed down the hurdles I came across in the process so you can get a head start on this.

**Keep in mind that Spring Boot 3 is to be released in __late November 2022__, so we are working with a milestone release (3.0.0-M5) here.**

## Let's get started
When we try to simply change the Spring Boot version to 3.0.0-M5 in our build.gradle file we get the following error:

```sh
- Plugin Repositories (could not resolve plugin artifact 'org.springframework.boot:org.springframework.boot.gradle.plugin:3.0.0-M5')
  Searched in the following repositories:
    maven(https://repo.spring.io/release)
    MavenRepo
    Gradle Central Plugin Repository
```

As Spring Boot 3.0.0-M5 is a milestone release, we need to include the milestone repository in our repositories section.

```groovy
repositories {
	maven { url 'https://repo.spring.io/milestone' }
  mavenCentral()
}
```

When you are using any Spring Boot plugins, you should also change the repositories for plugins in the settings.gradle file.
```sh
pluginManagement {
	repositories {
		maven { url 'https://repo.spring.io/milestone' }
		mavenCentral()
		gradlePluginPortal()
	}
}
```

So far so good, let's see what we get when we build our application.

```sh
Plugin [id: 'org.springframework.experimental.aot', version: '0.11.4'] was not found in any of the following sources:
```

Experimental? No longer! Since it's now in Spring Boot 3.0.0 we can get rid of this plugin.

Let's see where that gets us..

```sh
> Task :compileJava FAILED
error: package org.springframework.nativex.hint does not exist
```

Alright alright, nothing we can't handle. When building a Java application and compiling it with GraalVM we often need to play around with some GraalVM settings in the form of some CLI options that we need to pass along. 

Previously this could've been done by using @NativeHint annotations in your codebase, however this is no longer available in the Spring Boot 3 release. 

Luckily we can still modify our GraalVM build command depending on which build method your application uses. 

1. We can compile our application with GraalVM directly into a Docker image by using Paketo. 
2. We can use the [Native Build Tools](https://graalvm.github.io/native-build-tools/latest/index.html) to compile our application into a binary.

Either way there is still a way to provide those necessary build options to GraalVM. 

### Paketo
When using Paketo we can add the build options using an environment variable that is used during the build process:
```groovy
bootBuildImage {
	builder = 'paketobuildpacks/builder:base'
	environment = [
	   'BP_NATIVE_IMAGE': 'true',
	   'BP_NATIVE_IMAGE_BUILD_ARGUMENTS': '--initialize-at-build-time=ch.qos.logback,org.slf4j.impl.StaticLoggerBinder,org.slf4j.LoggerFactory,org.slf4j.MDC --initialize-at-run-time=io.netty.handler.ssl.BouncyCastleAlpnSslUtils' ]
}
```

### Native Build Tools
When using the Native Build Tools you can add it by using the [NativeImageOptions](https://graalvm.github.io/native-build-tools/latest/gradle-plugin.html#_native_image_options)

## Results

```sh
BUILD SUCCESSFUL in 4s
6 actionable tasks: 6 executed
```

One down, one to go! Let's try building our native image with `./gradlew clean bootBuildImage`.

```
Successfully built image 'docker.io/library/spring-boot-native-image:0.0.1-SNAPSHOT'

BUILD SUCCESSFUL in 2m 43s
6 actionable tasks: 6 executed
```

Look at that, we've got our Spring Boot 3.0.0-M5 project **successfully** built  into a GraalVM native image! 🎉

That's all folks 🤘 it was actually pretty easy..

