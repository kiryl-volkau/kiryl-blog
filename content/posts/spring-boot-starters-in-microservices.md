+++
date = '2025-07-10'
draft = false
title = 'Spring Boot Starters in Microservices: Sharing Code Without the Pain'
+++

# Spring Boot Starters in Microservices: Sharing Code Without the Pain
> _When you have 10 microservices that all need to talk to the same file storage API, do you copy-paste the client code 10 times? Or do you build a starter once and share it everywhere?_

In my [previous article](/posts/spring-boot-starters), we explored how Spring Boot starters work under the hood. Now let's talk about something more practical: **how to use starters to avoid code duplication in microservice architectures**.

When you're building microservices, you quickly run into a common problem: multiple services need the same functionality. Maybe they all need to call the same external API, or they all need the same testing setup, or they all need the same error handling.

The naive approach? Copy-paste the code into each service. But we all know how that story ends—you fix a bug in one service and forget to update the other nine.

The smart approach? **Build custom Spring Boot starters**.

## Two Types of Starters in Microservices

Not all starters are created equal. In a microservice architecture, you'll typically have two types:

### 1. Shared Starters (Used by Everyone)

These are starters that **every service** in your system needs. Think of them as your common foundation.

**Examples:**
- **Test starters** – common test utilities, base test classes, WireMock setup
- **Common utilities** – date mappers, file utilities, UUID generators
- **Error handling** – standardized exception handling across all services

### 2. Client Starters (Used by Specific Services)

These are starters that only **some services** need—specifically, services that need to communicate with a particular API.

**Examples:**
- **File API client** – only services that upload/download files need this
- **Search API client** – only services that perform search operations need this
- **Payment API client** – only services that process payments need this

The key insight: **client starters should be maintained by the team that owns the API**. If you own the File API, you should also own the file-api-client starter. This way, when your API changes, you update the client in one place, and all consuming services get the update.

## Real-World Example: File API Client Starter

Let's look at a concrete example. Imagine you have a File Storage API, and three different services need to upload files to it.

### The Old Way (Code Duplication)

Without a starter, each service would have its own Feign client:

```java
// In Service A
@FeignClient(name = "file-api", url = "${file.api.url}")
public interface FileClient {
    @PostMapping("/files/{bucket}/{dir}")
    FileResponse uploadFile(@PathVariable String bucket, 
                           @PathVariable String dir,
                           @RequestPart MultipartFile file);
}

// In Service B - same code, copy-pasted
@FeignClient(name = "file-api", url = "${file.api.url}")
public interface FileClient {
    // ... exact same methods
}

// In Service C - same code again
@FeignClient(name = "file-api", url = "${file.api.url}")
public interface FileClient {
    // ... you get the idea
}
```

Now imagine the File API team changes the endpoint structure. You need to update three different repositories. And if you have 10 services? Good luck.

### The Better Way (Client Starter)

Instead, create a **file-storage-client-starter** that all services can use:

**Step 1: Create the Feign Client**

```java
package com.company.clients.storage;

import org.springframework.cloud.openfeign.FeignClient;
import org.springframework.core.io.Resource;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;
import org.springframework.web.multipart.MultipartFile;

import static org.springframework.http.MediaType.MULTIPART_FORM_DATA_VALUE;

@FeignClient(
        name = "file-storage-api",
        url = "${storage.client.url:}",
        path = "${storage.client.path:/}",
        configuration = StorageClientConfiguration.class
)
public interface StorageApiClient {

    @PostMapping(value = "/files/{bucket}/{directory}", 
                 consumes = MULTIPART_FORM_DATA_VALUE)
    StoredFileDto uploadFile(
            @PathVariable("bucket") String bucket,
            @PathVariable("directory") String directory,
            @RequestPart("file") MultipartFile file
    );

    @GetMapping("/files/{bucket}/{objectId}?format=binary")
    ResponseEntity<Resource> downloadFile(
            @PathVariable("bucket") String bucket,
            @PathVariable("objectId") String objectId
    );

    @GetMapping("/files/{bucket}/{objectId}?format=meta")
    FileMetadataDto getFileMetadata(
            @PathVariable("bucket") String bucket,
            @PathVariable("objectId") String objectId
    );
}
```

**Step 2: Add Client Configuration**

```java
package com.company.clients.storage;

import feign.codec.Encoder;
import feign.form.spring.SpringFormEncoder;
import org.springframework.context.annotation.Bean;

public class StorageClientConfiguration {

    @Bean
    public Encoder feignFormEncoder() {
        return new SpringFormEncoder();
    }
}
```

**Step 3: Create Auto-Configuration**

```java
package com.company.clients.autoconfigure;

import org.springframework.boot.autoconfigure.AutoConfiguration;
import org.springframework.cloud.openfeign.EnableFeignClients;

@AutoConfiguration
@EnableFeignClients("com.company.clients")
public class StorageClientAutoConfiguration {
}
```

**Step 4: Register Auto-Configuration**

Create `META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports`:

```
com.company.clients.autoconfigure.StorageClientAutoConfiguration
```

**That's it!** Now any service can use the File API client by simply adding one dependency:

```xml
<dependency>
    <groupId>com.company</groupId>
    <artifactId>file-storage-client-starter</artifactId>
    <version>1.0.0</version>
</dependency>
```

The client is automatically configured and ready to use:

```java
@Service
public class DocumentService {
    
    private final StorageApiClient storageClient;
    
    public DocumentService(StorageApiClient storageClient) {
        this.storageClient = storageClient;
    }
    
    public void saveDocument(MultipartFile file) {
        storageClient.uploadFile("documents", "invoices", file);
    }
}
```

## Real-World Example: Shared Test Starter

Another common pattern is a **test starter** that provides common testing utilities for all services.

```java
package com.company.testing;

import com.fasterxml.jackson.databind.ObjectMapper;
import com.github.tomakehurst.wiremock.WireMockServer;
import lombok.SneakyThrows;
import org.junit.jupiter.api.extension.ExtendWith;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.autoconfigure.restdocs.AutoConfigureRestDocs;
import org.springframework.boot.test.autoconfigure.web.servlet.AutoConfigureMockMvc;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.core.io.ResourceLoader;
import org.springframework.test.context.junit.jupiter.SpringExtension;
import org.springframework.test.web.servlet.MockMvc;

import java.nio.file.Files;

import static org.springframework.boot.test.context.SpringBootTest.WebEnvironment.RANDOM_PORT;

@AutoConfigureMockMvc
@AutoConfigureRestDocs
@SpringBootTest(webEnvironment = RANDOM_PORT)
@ExtendWith(SpringExtension.class)
public abstract class BaseIntegrationTest {

    @Autowired(required = false)
    protected ObjectMapper objectMapper;
    
    @Autowired
    protected ResourceLoader resourceLoader;
    
    @Autowired
    protected MockMvc mockMvc;
    
    @Autowired(required = false)
    protected WireMockServer wireMockServer;

    @SneakyThrows
    protected String readClasspathFile(String path) {
        final var resource = resourceLoader.getResource(path);
        return Files.readString(resource.getFile().toPath());
    }

    @SneakyThrows
    protected byte[] readClasspathFileContent(String path) {
        final var resource = resourceLoader.getResource(path);
        return resource.getInputStream().readAllBytes();
    }
}
```

Now every service can write integration tests by simply extending this base class:

```java
public class DocumentControllerTest extends BaseIntegrationTest {
    
    @Test
    void shouldUploadDocument() throws Exception {
        String expectedResponse = readClasspathFile("classpath:responses/upload-success.json");
        
        mockMvc.perform(multipart("/documents")
                .file("file", "test content".getBytes()))
                .andExpect(status().isOk())
                .andExpect(content().json(expectedResponse));
    }
}
```

## The Monorepo Advantage

You might be thinking: "But we use a monorepo! All our services are in the same repository. Why do we need starters?"

Great question! Even in a monorepo, starters provide huge benefits:

### 1. No Versioning Headaches

In a monorepo, when you update the file-storage-client starter, all services that depend on it are updated in the same commit. No version conflicts, no "Service A uses v1.2 but Service B uses v1.5" problems.

### 2. Avoiding Code Duplication

Even though everything is in one repo, you still don't want to copy-paste the same Feign client code into 10 different services. When 2+ services need the same client, it's better to maintain **one starter** than duplicate the code.

### 3. Clear Ownership

Starters make it clear who owns what. The team that owns the File API also owns the file-storage-client starter. If you need to call the File API, you use their starter—you don't write your own client.

### 4. Easier Refactoring

When the File API changes, the File API team updates their starter. All consuming services get the update automatically (in a monorepo) or with a simple version bump (in separate repos).

## Key Takeaways

1. **Shared starters** (like test utilities) are used by all services and provide common foundation
2. **Client starters** (like API clients) are used only by services that need them
3. **Client starters should be owned by the API team** – if you own the API, you own the client
4. **Even in monorepos**, starters prevent code duplication and make ownership clear
5. **When 2+ services need the same functionality**, build a starter instead of copy-pasting

## Thanks for reading!

I hope this article showed you how starters can make your microservice architecture cleaner and more maintainable. If you have questions or want to share your own starter patterns, feel free to reach out!

