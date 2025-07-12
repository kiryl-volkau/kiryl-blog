+++
date = '2025-07-06T11:21:44-04:00'
draft = false
title = 'Spring Boot Starters'
+++

# Spring Boot Starters

When I first started working with Spring, I was always wondering:  
What is `spring-cloud-starter-openfeign`?  
What is `spring-boot-starter-web`?

At first glance, they just seemed like convenient libraries with some pre-made configuration. But actually, they are
what makes Spring Boot fast, easy to use, and help keep your `pom.xml` clean.

Early in my career, I didn’t think much about what was happening under the hood. I just added `spring-boot-starter-web`
and created REST controllers — without even asking myself how it really works.

In this post, I want to talk more about Spring Boot Starters:

- How they work
- Why they are valuable
- How to create your own Spring Boot Starter

## What is Spring Boot Starter ? 

Spring Boot Starters are ready-to-use sets of dependencies that you can quickly add to your project. They give you
everything you need to start working with Spring and related tools — without searching for examples or adding each
dependency manually.

For example, if you want to use Spring with JPA for database access, you just add the `spring-boot-starter-data-jpa` to
your project, and you’re ready to go.

### How do starters actually work?

Spring Boot Starters themselves don’t contain any logic — they are just dependency aggregators.

The real functionality comes from Spring Boot’s **auto-configuration mechanism**, which is enabled by default. When your application starts, Spring looks for classes listed in `META-INF/spring.factories` (or `spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports` in newer versions) and loads them based on the context.

### Before Auto-Configuration

Before Spring Boot, Java configuration was manual and repetitive:

* You had to define `DispatcherServlet`, set servlet mappings
* Configure `ViewResolver`, message converters, and `@EnableWebMvc`
* Register filters and listeners in `web.xml`

Auto-configuration eliminates all of this, using conditional logic and defaults. This is one of the reasons Spring Boot took off — it lets developers focus on business logic instead of boilerplate.

### What's inside `spring-boot-starter-web` (Spring Boot 3)

When you add [spring-boot-starter-web](https://github.com/spring-projects/spring-boot/blob/main/starter/spring-boot-starter-web/build.gradle):

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
    <version>3.5.3</version>
</dependency>
```

you automatically get the following dependencies:

- spring-boot-starter – the base starter (includes core infrastructure, logging, auto-configuration)
- spring-boot-starter-json – Jackson support for JSON serialization/deserialization
- spring-boot-starter-tomcat – embedded Tomcat servlet container
- spring-web – core HTTP functionality and low-level client/server APIs
- spring-webmvc – Spring MVC framework: DispatcherServlet, controller mapping, routing

### Example: [DispatcherServletAutoConfiguration](https://github.com/spring-projects/spring-boot/blob/1758f509bce9d4f5b32a9bd1c8428738cc1ae751/module/spring-boot-webmvc/src/main/java/org/springframework/boot/webmvc/autoconfigure/DispatcherServletAutoConfiguration.java#L69)

A core component of the Spring MVC framework is the `DispatcherServlet`, which handles incoming HTTP requests. When you include `spring-boot-starter-web`, Spring Boot automatically registers a `DispatcherServlet` bean and configures it to respond to all incoming requests.

This is handled by the class `DispatcherServletAutoConfiguration`, located in the `spring-boot-autoconfigure` module.

```java
@AutoConfigureOrder(Ordered.HIGHEST_PRECEDENCE)
@AutoConfiguration
@ConditionalOnWebApplication(type = Type.SERVLET)
@ConditionalOnClass(DispatcherServlet.class)
public class DispatcherServletAutoConfiguration {
    // ...
}
```

Let’s break down these annotations in more detail:

* `@AutoConfiguration`

    * Marks the class as an auto-configuration candidate.
    * Spring Boot uses this annotation (instead of just `@Configuration`) to include the class via `spring.factories` or `AutoConfiguration.imports`.

* `@AutoConfigureOrder(Ordered.HIGHEST_PRECEDENCE)`

    * Controls the execution order of auto-configurations.
    * Spring uses constants like:

        * `Ordered.HIGHEST_PRECEDENCE` (runs first)
        * `Ordered.LOWEST_PRECEDENCE` (runs last)
        * Or any custom integer value
    * This ordering matters when you have overlapping beans or you want something to happen early.

* `@ConditionalOnWebApplication(type = Type.SERVLET)`

    * Ensures the configuration only applies to traditional servlet-based applications.
    * Type options:

        * `SERVLET` – classic Spring MVC
        * `REACTIVE` – Spring WebFlux
        * `ANY` – applies to either
    * This type system is an **internal DSL** Spring Boot uses to restrict configurations to proper environments.

* `@ConditionalOnClass(DispatcherServlet.class)`

    * Activates the configuration only if the specified class exists on the classpath.
    * Spring Boot has many such conditional annotations:

        * `@ConditionalOnMissingBean`
        * `@ConditionalOnProperty`
        * `@ConditionalOnResource`
        * `@ConditionalOnBean`
    * This makes configuration declarative and environment-aware.

These annotations represent Spring Boot's **declarative and conditional approach** to configuration. Instead of forcing developers to think about what to include or wire, Spring Boot makes intelligent decisions based on your environment and classpath.

---
## What’s Next?

So far we've looked at how existing starters work. In the next article, we'll explore how to create your own Spring Boot Starter — for example, a custom Feign client starter that auto-registers external service clients with preconfigured settings.

Topics we’ll cover:

* How to structure your starter project
* How to use `@AutoConfiguration` to register your beans
* How to use `@EnableFeignClients` in the right context
* How to publish and reuse the starter internally or publicly

## Thanks for reading!

Thank you! I hope this post gave you an understanding of how Spring Boot starters actually work under the hood.

Feel free to share your thoughts or questions — and don't forget to check back for Part 2, where we'll build our own starter from scratch.


