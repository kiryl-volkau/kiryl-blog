+++
date = '2025-11-20'
draft = false
title = 'WebClient Configuration: Stop Copy-Pasting Your HTTP Clients'
+++

# WebClient Configuration: Stop Copy-Pasting Your HTTP Clients

> _You have 5 external APIs to call. Do you configure logging, retries, and timeouts 5 times? Or do you set it up once and reuse it everywhere?_

Here's a scenario I see all the time: you're building a microservice that calls multiple external APIs—a payment gateway, a file storage service, a notification API. Each needs its own `WebClient`, and each needs logging, error handling, retries, and timeouts.

So what do most developers do? 

**They copy-paste the same configuration code for each client.**

And then when you need to change the retry logic or add request logging, you're updating it in 5 different places.

There's a better way.

---

## The Problem: Code Duplication Everywhere

Without a base configuration, your code looks like this:

```java
@Configuration
@Slf4j
public class WebClientConfig {

    @Bean
    public WebClient paymentClient() {
        return WebClient.builder()
                .baseUrl("https://api.payment.com")
                .defaultHeader("User-Agent", "MyApp/1.0")
                .filter(logRequest())
                .filter(logResponse())
                .filter(retryFilter())
                .build();
    }

    @Bean
    public WebClient storageClient() {
        return WebClient.builder()
                .baseUrl("https://api.storage.com")
                .defaultHeader("User-Agent", "MyApp/1.0")  // duplicated
                .filter(logRequest())  // duplicated
                .filter(logResponse())  // duplicated
                .filter(retryFilter())  // duplicated
                .build();
    }

    @Bean
    public WebClient notificationClient() {
        return WebClient.builder()
                .baseUrl("https://api.notifications.com")
                .defaultHeader("User-Agent", "MyApp/1.0")  // duplicated again!
                .filter(logRequest())  // duplicated again!
                .filter(logResponse())  // duplicated again!
                .filter(retryFilter())  // duplicated again!
                .build();
    }

    // Filter implementations...
}
```

**The pain points:**

- Every new API = 6 lines of duplicated configuration
- Need to add error handling? Update **every single bean**
- Change retry logic? Hunt through multiple methods
- By the time you have 5-6 APIs, you're maintaining dozens of duplicated lines

---

## The Solution: Base Builder Pattern

Here's the key insight: **`WebClient.Builder` is cloneable**.

You can create a base builder with all your common configuration, then clone it for each specific client.

### Step 1: Create a Base Builder

```java
import lombok.extern.slf4j.Slf4j;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.web.reactive.function.client.ExchangeFilterFunction;
import org.springframework.web.reactive.function.client.WebClient;
import reactor.core.publisher.Mono;
import reactor.util.retry.Retry;
import java.time.Duration;

@Configuration
@Slf4j
public class BaseWebClientConfig {

    @Bean
    public WebClient.Builder baseWebClientBuilder() {
        return WebClient.builder()
                .defaultHeader("User-Agent", "MyApp/1.0")
                .defaultHeader("Accept", "application/json")
                .filter(logRequest())
                .filter(logResponse())
                .filter(retryWithBackoff())
                .filter(mapCommonErrors());
    }

    private ExchangeFilterFunction logRequest() {
        return ExchangeFilterFunction.ofRequestProcessor(request -> {
            log.info("Request: {} {}", request.method(), request.url());
            return Mono.just(request);
        });
    }

    private ExchangeFilterFunction logResponse() {
        return ExchangeFilterFunction.ofResponseProcessor(response -> {
            log.info("Response: {}", response.statusCode());
            return Mono.just(response);
        });
    }

    private ExchangeFilterFunction retryWithBackoff() {
        return (request, next) -> next.exchange(request)
                .retryWhen(Retry.backoff(3, Duration.ofSeconds(1))
                        .maxBackoff(Duration.ofSeconds(10))
                        .filter(throwable -> {
                            // Only retry on network errors and 5xx
                            if (throwable instanceof WebClientRequestException) {
                                return true;
                            }
                            if (throwable instanceof WebClientResponseException) {
                                return ((WebClientResponseException) throwable)
                                        .getStatusCode().is5xxServerError();
                            }
                            return false;
                        }));
    }

    private ExchangeFilterFunction mapCommonErrors() {
        return ExchangeFilterFunction.ofResponseProcessor(response -> {
            if (response.statusCode().is5xxServerError()) {
                return response.bodyToMono(String.class)
                        .defaultIfEmpty("Unknown error")
                        .flatMap(body -> Mono.error(
                            new ExternalApiException("Server error: " + body)));
            }
            return Mono.just(response);
        });
    }
}
```

### Step 2: Clone for Specific Clients

Now when you need a client, **clone** the base builder:

```java
@Configuration
public class ApiClientsConfig {

    @Bean
    public WebClient paymentClient(WebClient.Builder baseWebClientBuilder) {
        return baseWebClientBuilder
                .clone()  // Inherits all base configuration!
                .baseUrl("https://api.payment.com")
                .defaultHeader("X-API-Key", "${payment.api.key}")
                .build();
    }

    @Bean
    public WebClient storageClient(WebClient.Builder baseWebClientBuilder) {
        return baseWebClientBuilder
                .clone()  // Inherits all base configuration!
                .baseUrl("https://api.storage.com")
                .defaultHeader("Authorization", "Bearer ${storage.token}")
                .build();
    }

    @Bean
    public WebClient notificationClient(WebClient.Builder baseWebClientBuilder) {
        return baseWebClientBuilder
                .clone()  // Inherits all base configuration!
                .baseUrl("https://api.notifications.com")
                .defaultHeader("X-Service-Key", "${notification.key}")
                .build();
    }
}
```

**What just happened?**

✅ Each client gets logging, retries, and error handling automatically  
✅ No code duplication  
✅ Change retry logic once, all clients update  
✅ Add a new filter? One place to update  

---

## The Single Responsibility Principle for Filters

Before we go further, let's talk about an important principle: **each filter should do ONE thing**.

### ❌ Don't Do This

```java
// BAD: One filter doing everything
private ExchangeFilterFunction doEverything() {
    return (request, next) -> {
        log.info("Request: {}", request.url());  // Logging
        
        return next.exchange(request)
                .flatMap(response -> {
                    log.info("Response: {}", response.statusCode());  // More logging
                    
                    if (response.statusCode().is5xxServerError()) {  // Error handling
                        return Mono.error(new ExternalApiException("Error!"));
                    }
                    return Mono.just(response);
                })
                .retryWhen(Retry.backoff(3, Duration.ofSeconds(1)));  // Retry logic
    };
}
```

This is a maintenance nightmare.

### ✅ Do This Instead

```java
// GOOD: Each filter has one responsibility
.filter(logRequest())        // Only logs requests
.filter(logResponse())       // Only logs responses
.filter(retryWithBackoff())  // Only handles retries
.filter(mapCommonErrors())   // Only maps errors
```

**Why?**

- **Easier to modify**: Change retry logic without touching logging
- **Easier to test**: Test each filter independently
- **Easier to disable**: Comment out logging without affecting error handling
- **Easier to understand**: Each filter's purpose is crystal clear

---

## Understanding Filter Execution Order

Filters execute in a specific order. Understanding this is crucial:

{{< mermaid >}}
graph TB
    subgraph "Request Phase (Top to Bottom)"
        A[1. logRequest] --> B[2. addHeaders]
        B --> C[3. Execute HTTP Request]
    end

    subgraph "Response Phase (Bottom to Top)"
        C --> D[3. mapErrors]
        D --> E[2. logResponse]
        E --> F[1. Return to caller]
    end

    style A fill:#e1f5e1
    style B fill:#e1f5e1
    style D fill:#ffe1e1
    style E fill:#ffe1e1
{{< /mermaid >}}

**Key insight**: Request filters execute top-to-bottom, response filters execute bottom-to-top.

This means:
- Request is logged **before** headers are added
- Errors are mapped **before** response is logged

---

## ⚠️ Critical Warning: Retry + POST = Danger

Here's a gotcha that can cost you money:

> **⚠️ Warning**: Never retry POST requests without idempotency protection!

```java
// DANGEROUS: This can duplicate payments!
@Bean
public WebClient paymentClient(WebClient.Builder baseWebClientBuilder) {
    return baseWebClientBuilder
            .clone()  // Inherits retry logic
            .baseUrl("https://api.payment.com")
            .build();
}

// If the payment succeeds but the response times out,
// the retry will charge the customer TWICE!
```

**The fix**: Add idempotency keys to POST requests. We'll cover this in the next article.

---

## Key Takeaways

1. **Use `WebClient.Builder.clone()`** to reuse base configuration
2. **One filter = one responsibility** (logging, retry, error handling separate)
3. **Filters execute in order**: request top-to-bottom, response bottom-to-top
4. **Be careful with retries**: POST + retry = potential duplicates
5. **Update once, apply everywhere**: Change base builder, all clients update

---

## What's Next?

In the next article, we'll cover production-ready patterns:

- **Idempotency keys** for safe POST retries
- **Timeout configuration** and overrides
- **Testing** your filters independently
- **Common pitfalls** and how to avoid them

---

## Thanks for reading!

The base builder pattern is simple but powerful. Use it to keep your HTTP clients DRY and maintainable.

**Coming up next**: Advanced WebClient patterns for production systems.

