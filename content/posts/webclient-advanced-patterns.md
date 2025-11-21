+++
date = '2025-11-21'
draft = false
title = 'Advanced WebClient Patterns: Idempotency, Timeouts, and Testing'
+++

# Advanced WebClient Patterns: Idempotency, Timeouts, and Testing

> _In the [previous article](/posts/webclient-base-builder-pattern), we covered the base builder pattern for WebClient. Now let's make it production-ready._

You've set up a base `WebClient.Builder` with logging, retries, and error handling. Great start!

But production systems need more:

- **Idempotency keys** to prevent duplicate payments
- **Custom timeouts** for slow operations
- **Service-specific error handling** for business logic
- **Tests** to verify everything works

Let's dive in.

---

## Quick Recap: The Base Builder

In the previous article, we created a base builder:

```java
@Bean
public WebClient.Builder baseWebClientBuilder() {
    return WebClient.builder()
            .defaultHeader("User-Agent", "MyApp/1.0")
            .filter(logRequest())
            .filter(logResponse())
            .filter(retryWithBackoff())
            .filter(mapCommonErrors());
}
```

Now we'll add service-specific filters on top of this base.

---

## Real-World Example: Payment Gateway Client

Let's build a production-ready payment client. Requirements:

- ✅ API key authentication
- ✅ Merchant ID in headers
- ✅ **Idempotency keys** for POST requests (prevent duplicate charges)
- ✅ **Payment-specific errors** (insufficient funds, card declined)

### The Implementation

```java
import lombok.extern.slf4j.Slf4j;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.http.HttpMethod;
import org.springframework.web.reactive.function.client.ClientRequest;
import org.springframework.web.reactive.function.client.ExchangeFilterFunction;
import org.springframework.web.reactive.function.client.WebClient;
import reactor.core.publisher.Mono;
import java.util.UUID;

@Configuration
@Slf4j
public class PaymentGatewayConfig {

    @Value("${payment.gateway.url}")
    private String paymentGatewayUrl;

    @Value("${payment.gateway.api-key}")
    private String apiKey;

    @Value("${payment.gateway.merchant-id}")
    private String merchantId;

    @Bean
    public WebClient paymentGatewayClient(WebClient.Builder baseWebClientBuilder) {
        return baseWebClientBuilder
                .clone()  // Inherits: logging, retries, common errors
                .baseUrl(paymentGatewayUrl)
                .defaultHeader("X-API-Key", apiKey)
                .defaultHeader("X-Merchant-ID", merchantId)
                .filter(addIdempotencyKey())      // Payment-specific
                .filter(mapPaymentErrors())        // Payment-specific
                .build();
    }

    // Filter 1: Add idempotency key to POST requests
    private ExchangeFilterFunction addIdempotencyKey() {
        return ExchangeFilterFunction.ofRequestProcessor(request -> {
            if (request.method() == HttpMethod.POST) {
                String idempotencyKey = UUID.randomUUID().toString();
                log.debug("Adding idempotency key: {}", idempotencyKey);
                
                return Mono.just(ClientRequest.from(request)
                        .header("Idempotency-Key", idempotencyKey)
                        .build());
            }
            return Mono.just(request);
        });
    }

    // Filter 2: Map payment-specific errors
    private ExchangeFilterFunction mapPaymentErrors() {
        return ExchangeFilterFunction.ofResponseProcessor(response -> {
            int status = response.statusCode().value();

            if (status == 402) {  // Payment Required
                return response.bodyToMono(String.class)
                        .defaultIfEmpty("Payment required")
                        .flatMap(body -> {
                            log.error("Payment failed - insufficient funds: {}", body);
                            return Mono.error(new InsufficientFundsException(body));
                        });
            }

            if (status == 422) {  // Unprocessable Entity
                return response.bodyToMono(String.class)
                        .defaultIfEmpty("Unprocessable entity")
                        .flatMap(body -> {
                            log.error("Payment failed - card declined: {}", body);
                            return Mono.error(new CardDeclinedException(body));
                        });
            }

            return Mono.just(response);  // Let base error handler deal with others
        });
    }
}
```

### Using the Payment Client

```java
@Service
@Slf4j
public class PaymentService {

    private final WebClient paymentGatewayClient;

    public PaymentService(WebClient paymentGatewayClient) {
        this.paymentGatewayClient = paymentGatewayClient;
    }

    public Mono<PaymentResponse> processPayment(PaymentRequest request) {
        return paymentGatewayClient
                .post()
                .uri("/payments")
                .bodyValue(request)
                .retrieve()
                .bodyToMono(PaymentResponse.class)
                .doOnSuccess(response -> 
                    log.info("Payment processed: {}", response.getTransactionId()))
                .doOnError(InsufficientFundsException.class, ex ->
                    log.error("Payment failed - insufficient funds"));
    }
}
```

### How Filters Execute

Here's what happens when you call `processPayment()`:

{{< mermaid >}}
sequenceDiagram
    participant S as Service
    participant F1 as addIdempotencyKey
    participant F2 as logRequest
    participant F3 as HTTP Request
    participant F4 as logResponse
    participant F5 as retryWithBackoff
    participant F6 as mapPaymentErrors
    participant F7 as mapCommonErrors

    S->>F1: POST /payments
    Note over F1: Add Idempotency-Key header
    F1->>F2: Modified request
    Note over F2: Log request
    F2->>F3: Execute HTTP
    F3->>F4: Response received
    Note over F4: Log response
    F4->>F5: Check if retry needed
    F5->>F6: Map payment errors (402, 422)
    F6->>F7: Map common errors (4xx, 5xx)
    F7->>S: Return result or exception
{{< /mermaid >}}

**Key insight**: Idempotency key is added **before** logging, so the log shows the actual key sent.

---

## Timeout Configuration

Different APIs need different timeouts. File processing might take 5 minutes, but a payment should respond in seconds.

### Base Builder with Timeouts

```java
import io.netty.channel.ChannelOption;
import io.netty.handler.timeout.ReadTimeoutHandler;
import io.netty.handler.timeout.WriteTimeoutHandler;
import org.springframework.http.client.reactive.ReactorClientHttpConnector;
import reactor.netty.http.client.HttpClient;

@Bean
public WebClient.Builder baseWebClientBuilder() {
    HttpClient httpClient = HttpClient.create()
            .option(ChannelOption.CONNECT_TIMEOUT_MILLIS, 5000)  // 5s connect
            .responseTimeout(Duration.ofSeconds(10))              // 10s response
            .doOnConnected(conn -> conn
                    .addHandlerLast(new ReadTimeoutHandler(10))   // 10s read
                    .addHandlerLast(new WriteTimeoutHandler(10))); // 10s write

    return WebClient.builder()
            .clientConnector(new ReactorClientHttpConnector(httpClient))
            .defaultHeader("User-Agent", "MyApp/1.0")
            .filter(logRequest())
            .filter(logResponse())
            .filter(retryWithBackoff())
            .filter(mapCommonErrors());
}
```

### Override Timeout for Slow Operations

```java
@Bean
public WebClient fileProcessingClient(WebClient.Builder baseWebClientBuilder) {
    HttpClient slowHttpClient = HttpClient.create()
            .responseTimeout(Duration.ofMinutes(5));  // 5 minute timeout

    return baseWebClientBuilder
            .clone()  // Still inherits all filters!
            .clientConnector(new ReactorClientHttpConnector(slowHttpClient))
            .baseUrl("https://api.file-processing.com")
            .defaultHeader("Authorization", "Bearer ${file.token}")
            .build();
}
```

> **💡 Tip**: Timeout configuration is separate from filters. Overriding the HTTP client doesn't affect your filter chain.

---

## Testing Your Filters

One huge benefit of separating filters: **you can test each one independently**.

### Example: Testing Idempotency Filter

```java
import org.junit.jupiter.api.Test;
import org.springframework.http.HttpMethod;
import org.springframework.web.reactive.function.client.ClientRequest;
import org.springframework.web.reactive.function.client.ExchangeFunction;
import reactor.core.publisher.Mono;
import reactor.test.StepVerifier;

import java.net.URI;

import static org.assertj.core.api.Assertions.assertThat;
import static org.mockito.Mockito.mock;

class IdempotencyFilterTest {

    @Test
    void shouldAddKeyToPostRequests() {
        // Arrange
        ExchangeFilterFunction filter = addIdempotencyKey();
        ExchangeFunction mockExchange = mock(ExchangeFunction.class);
        
        ClientRequest request = ClientRequest
                .create(HttpMethod.POST, URI.create("http://test.com"))
                .build();
        
        // Act & Assert
        StepVerifier.create(filter.filter(request, mockExchange))
                .assertNext(modifiedRequest -> {
                    ClientRequest req = (ClientRequest) modifiedRequest;
                    assertThat(req.headers()).containsKey("Idempotency-Key");
                })
                .verifyComplete();
    }
    
    @Test
    void shouldNotAddKeyToGetRequests() {
        // Arrange
        ExchangeFilterFunction filter = addIdempotencyKey();
        ExchangeFunction mockExchange = mock(ExchangeFunction.class);
        
        ClientRequest request = ClientRequest
                .create(HttpMethod.GET, URI.create("http://test.com"))
                .build();
        
        // Act & Assert
        StepVerifier.create(filter.filter(request, mockExchange))
                .assertNext(modifiedRequest -> {
                    ClientRequest req = (ClientRequest) modifiedRequest;
                    assertThat(req.headers()).doesNotContainKey("Idempotency-Key");
                })
                .verifyComplete();
    }
}
```

**Why this matters:**

✅ Test filters without making actual HTTP calls  
✅ Verify behavior for different HTTP methods  
✅ When a filter breaks, you know exactly which one  

---

## Common Pitfalls and How to Avoid Them

### 1. ⚠️ Logging Request/Response Bodies

```java
// DANGEROUS: This breaks reactive streams!
private ExchangeFilterFunction logRequestBody() {
    return ExchangeFilterFunction.ofRequestProcessor(request -> {
        return request.body()  // This consumes the body!
                .doOnNext(body -> log.info("Body: {}", body))
                .then(Mono.just(request));
    });
}
```

**The problem**: You can only read a reactive stream once. After logging, the body is gone.

**The fix**: Use `ExchangeStrategies` to buffer the body, or don't log it at all.

> **💡 Tip**: For debugging, use a proxy like Charles or Wireshark instead of logging bodies in code.

### 2. ⚠️ Too Many Filters

```java
// HARD TO DEBUG: 10+ filters in the chain
.filter(logRequest())
.filter(addCorrelationId())
.filter(addTraceId())
.filter(addUserAgent())
.filter(addAcceptHeader())
.filter(addContentType())
.filter(validateRequest())
.filter(retryWithBackoff())
.filter(circuitBreaker())
.filter(rateLimiting())
.filter(mapErrors())
.filter(logResponse())
```

**The problem**: When something breaks, good luck figuring out which filter caused it.

**The fix**: Combine related concerns. Headers can be set with `.defaultHeader()` instead of filters.

### 3. ⚠️ Retry Without Idempotency

We mentioned this in the previous article, but it's worth repeating:

```java
// DANGEROUS: Can duplicate orders/payments!
public Mono<OrderResponse> createOrder(OrderRequest request) {
    return webClient
            .post()
            .uri("/orders")
            .bodyValue(request)
            .retrieve()
            .bodyToMono(OrderResponse.class);
    // If this times out after the order is created,
    // the retry will create a SECOND order!
}
```

**The fix**: Always add idempotency keys to POST/PUT/PATCH requests that modify state.

> **⚠️ Warning**: GET and DELETE are naturally idempotent. POST, PUT, and PATCH are NOT.

---

## When to Extract to a Starter

Should you extract your base WebClient configuration into a custom Spring Boot starter?

### ✅ Extract to a Starter If:

1. **You have 3+ microservices** that all call external APIs
2. **You want consistent behavior** across all services (same logging, retry logic, error handling)
3. **You have company-wide standards** for HTTP client configuration
4. **You're tired of copy-pasting** the base configuration into every new service

### ❌ Don't Extract If:

1. **You only have 1-2 services** – just keep the configuration in your service
2. **Each service has wildly different requirements** – the base configuration won't help much
3. **You're still experimenting** – wait until the pattern stabilizes

### The Honest Truth

I've seen teams go both ways. Some extract too early and end up with a starter nobody uses. Others wait too long and waste time copy-pasting.

**My rule of thumb**: If you're about to copy-paste the same WebClient configuration into a **third service**, that's when you extract it to a starter.

---

## Visualizing the Request-Response Flow

Here's how a request flows through your filter chain:

{{< mermaid >}}
graph LR
    subgraph "Your Service"
        A[Call WebClient]
    end

    subgraph "Request Filters (Top to Bottom)"
        B[1. Add Idempotency Key]
        C[2. Log Request]
    end

    subgraph "HTTP"
        D[Execute Request]
    end

    subgraph "Response Filters (Bottom to Top)"
        E[3. Retry if 5xx]
        F[4. Map Payment Errors]
        G[5. Map Common Errors]
        H[6. Log Response]
    end

    subgraph "Your Service"
        I[Handle Result]
    end

    A --> B --> C --> D
    D --> E --> F --> G --> H --> I

    style B fill:#e1f5e1
    style C fill:#e1f5e1
    style E fill:#ffe1e1
    style F fill:#ffe1e1
    style G fill:#ffe1e1
    style H fill:#ffe1e1
{{< /mermaid >}}

**Key insight**: Request filters modify the request before it's sent. Response filters handle the response after it's received.

---

## Key Takeaways

1. **Add idempotency keys** to POST requests to prevent duplicates
2. **Configure timeouts** based on the operation (fast vs. slow APIs)
3. **Test filters independently** – they're pure functions
4. **Don't log request/response bodies** – it breaks reactive streams
5. **Keep filter chains short** – 4-6 filters is plenty
6. **Extract to a starter** when you have 3+ services with similar needs
7. **Service-specific filters** go on top of base filters (payment errors, rate limiting, etc.)

---

## What We've Covered

In this two-part series, we've built a production-ready WebClient configuration:

**Part 1**: Base builder pattern with logging, retries, and error handling
**Part 2**: Idempotency, timeouts, testing, and common pitfalls

You now have everything you need to build robust HTTP clients in Spring Boot microservices.

---

## Thanks for reading!

The patterns in this article come from real production systems. Use them to build reliable, maintainable HTTP clients.

**Questions or feedback?** Let me know what patterns you're using in your projects!



