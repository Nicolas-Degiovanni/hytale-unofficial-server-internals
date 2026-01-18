---
description: Architectural reference for SessionServiceClient
---

# SessionServiceClient

**Package:** com.hypixel.hytale.server.core.auth
**Type:** Transient / Service

## Definition
```java
// Signature
public class SessionServiceClient {
```

## Architecture & Concepts
The **SessionServiceClient** is a critical component in the authentication and session management subsystem. It functions as a stateless, client-side gateway that abstracts all direct HTTP communication with Hytale's backend authentication and account services. Its primary architectural role is to decouple the core game logic (both client and server) from the specifics of the remote API endpoints, protocols, and data formats.

This class is designed for a non-blocking, asynchronous environment. Every network operation returns a **CompletableFuture**, ensuring that calling threads, such as the main server tick loop or the client render thread, are not stalled waiting for network I/O. It exclusively uses the modern Java 11 **HttpClient** for efficient connection management and request execution.

Serialization and deserialization of JSON payloads are handled internally by the Hytale **Codec** system, providing a type-safe boundary between the raw text-based API and the engine's Java objects.

## Lifecycle & Ownership
- **Creation:** An instance of **SessionServiceClient** is created during the application's bootstrap sequence. The constructor requires the base URL of the session service, which is typically loaded from a configuration file specific to the deployment environment (e.g., production, staging). It is expected to be instantiated by a central service registry or dependency injection framework.

- **Scope:** The object is designed to be long-lived, persisting for the entire duration of the application's runtime. A single, shared instance should be used for all session-related API calls to leverage the underlying **HttpClient**'s connection pooling and resource management.

- **Destruction:** The class does not implement any explicit cleanup logic like a *close* method. It is subject to standard Java garbage collection when its owner (e.g., the root service container) is shut down.

## Internal State & Concurrency
- **State:** The internal state consists of the final fields **httpClient** and **sessionServiceUrl**, which are set at construction and never modified. The class is therefore stateless concerning application data; it does not cache tokens, profiles, or session information. Each method call is an independent, self-contained operation.

- **Thread Safety:** This class is fully thread-safe. The underlying **HttpClient** is designed for concurrent use. Since the internal state is immutable and each method invocation creates new, request-specific objects, a single **SessionServiceClient** instance can be safely shared and invoked from any thread without external synchronization.

## API Surface
The public API represents a series of operations in the user authentication and session lifecycle.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| requestAuthorizationGrantAsync(...) | CompletableFuture<String> | Network I/O | Initiates the server join flow by exchanging an identity token for a short-lived authorization grant. |
| exchangeAuthGrantForTokenAsync(...) | CompletableFuture<String> | Network I/O | Completes the server join flow by exchanging the authorization grant for a final access token. |
| getJwks() | JwksResponse | Network I/O | Fetches the JSON Web Key Set from the well-known endpoint, used for validating JWT signatures. |
| getGameProfiles(...) | GameProfile[] | Network I/O | Retrieves the game profiles associated with a user's master account using an OAuth token. |
| createGameSession(...) | GameSessionResponse | Network I/O | Creates a new, persistent game session for a selected user profile. |
| refreshSessionAsync(...) | CompletableFuture<GameSessionResponse> | Network I/O | Refreshes an existing game session to extend its lifetime, returning a new set of tokens. |
| terminateSession(...) | void | Network I/O | Invalidates a game session token, effectively logging the user out from the backend. This is a fire-and-forget operation. |

## Integration Patterns

### Standard Usage
The client should be retrieved from a central service provider. Operations should be performed asynchronously by chaining callbacks to the returned **CompletableFuture** to avoid blocking critical threads.

```java
// Correctly use the async API to avoid blocking
SessionServiceClient authClient = serviceRegistry.get(SessionServiceClient.class);

authClient.refreshSessionAsync(currentToken)
    .thenAccept(newSession -> {
        // This code executes on a worker thread when the network call completes
        if (newSession != null) {
            System.out.println("Session successfully refreshed. New identity token: " + newSession.identityToken);
            // Update the application's current session state here
        } else {
            System.err.println("Failed to refresh session.");
        }
    })
    .exceptionally(ex -> {
        // Handle network errors or other exceptions
        LOGGER.log("Exception during session refresh: " + ex.getMessage());
        return null; // Required for exceptionally block
    });
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not call `new SessionServiceClient()` repeatedly for each request. This is highly inefficient as it bypasses connection pooling and creates a new **HttpClient** instance and thread pool every time. Use a single, shared instance.

- **Blocking on Futures:** Do not call **.get()** or **.join()** on the returned **CompletableFuture** from a performance-critical thread (like the main game loop). This will freeze the application until the network request completes, leading to severe performance degradation and unresponsiveness.

- **Ignoring Failures:** The methods can return null within the **CompletableFuture** to indicate an API-level failure (e.g., HTTP 404, invalid response body). Code must be robust and handle these null cases gracefully instead of assuming success.

## Data Pipeline
The **SessionServiceClient** acts as a marshalling layer, converting between Java types and the JSON-based HTTP API.

> Flow:
> Method Call (Java types) -> Internal JSON String Builder -> **SessionServiceClient** (HttpClient) -> HTTP POST/GET Request -> HTTP Response (JSON String) -> Hytale Codec Deserializer -> **SessionServiceClient** -> CompletableFuture (Java DTO) -> Application Logic

