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

## What is Spring Boot Starer ? 

Spring Boot Starters are ready-to-use sets of dependencies that you can quickly add to your project. They give you
everything you need to start working with Spring and related tools — without searching for examples or adding each
dependency manually.

For example, if you want to use Spring with JPA for database access, you just add the `spring-boot-starter-data-jpa` to
your project, and you’re ready to go. 



https://docs.spring.io/spring-boot/reference/using/build-systems.html#using.build-systems.starters
https://github.com/spring-projects/spring-boot/tree/main/spring-boot-project/spring-boot-starters



