---
description: Architectural reference for MemoryAuthCredentialStoreProvider
---

# MemoryAuthCredentialStoreProvider

**Package:** com.hypixel.hytale.server.core.auth
**Type:** Utility

## Definition
```java
// Signature
public class MemoryAuthCredentialStoreProvider implements AuthCredentialStoreProvider {
```

## Architecture & Concepts
The MemoryAuthCredentialStoreProvider is a concrete implementation of the AuthCredentialStoreProvider strategy interface. Its sole responsibility is to act as a factory for creating a non-persistent, in-memory authentication credential store.

This provider represents the simplest storage strategy within the server's authentication framework. It is identified by the static ID "Memory", which allows server administrators to select it via configuration files. The presence of a static BuilderCodec field indicates that the server's configuration and dependency injection system is responsible for its instantiation, not developers directly.

This class decouples the server's core authentication logic from the specifics of credential storage. By selecting this provider, the server is configured to operate in a mode where authentication credentials are not persisted across restarts, making it ideal for development, testing, or specific single-session game modes.

## Lifecycle & Ownership
-   **Creation:** Instantiated by the server's configuration loading system during the bootstrap phase. The system uses the public static CODEC to deserialize the configuration and create an instance of this provider when the authentication store type is set to "Memory".
-   **Scope:** The provider object itself is typically held by a central authentication service manager for the duration of the server's runtime. However, its primary purpose is fulfilled at creation time when its createStore method is invoked.
-   **Destruction:** The object is eligible for garbage collection upon server shutdown when the services holding a reference to it are dismantled. It manages no external resources and requires no explicit cleanup.

## Internal State & Concurrency
-   **State:** This class is **stateless and immutable**. It contains no instance fields and its behavior is constant.
-   **Thread Safety:** This class is inherently **thread-safe**. The createStore method can be invoked concurrently from multiple threads without risk of race conditions, as it simply allocates and returns a new, independent DefaultAuthCredentialStore object on each call.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| createStore() | IAuthCredentialStore | O(1) | Factory method that instantiates and returns a new DefaultAuthCredentialStore. |
| ID | static String | O(1) | The unique configuration key for this provider. |
| CODEC | static BuilderCodec | O(1) | The mechanism for configuration-driven instantiation of the provider. |

## Integration Patterns

### Standard Usage
This class is not intended for direct use by developers. It is designed to be specified within a server configuration file and managed by the server's authentication service. The system internally resolves the "Memory" identifier to this provider.

A conceptual example of a server configuration might look like this:
```yaml
# server-config.yml
services:
  authentication:
    store:
      type: "Memory" # This string ID resolves to MemoryAuthCredentialStoreProvider
      # No further options are needed for the memory provider
```

The server's bootstrap logic would then use this configuration to create the appropriate store.

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Do not call `new MemoryAuthCredentialStoreProvider()`. The server's architecture relies on the static CODEC for proper initialization and integration into the service framework.
-   **Production Usage:** Using this provider in a production environment that requires user accounts to persist between server restarts is a critical misconfiguration. All authentication data will be lost upon shutdown. This provider is strictly for ephemeral sessions.

## Data Pipeline
This provider is not part of a runtime data processing pipeline. Instead, it is a key component in the server's *configuration and initialization pipeline*.

> Flow:
> Server Configuration File -> Configuration Service -> **MemoryAuthCredentialStoreProvider.CODEC** -> Provider Instance -> `createStore()` -> `DefaultAuthCredentialStore` -> Injected into Auth Service

