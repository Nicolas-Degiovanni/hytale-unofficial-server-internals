---
description: Architectural reference for AuthCredentialStoreProvider
---

# AuthCredentialStoreProvider

**Package:** com.hypixel.hytale.server.core.auth
**Type:** Provider Interface

## Definition
```java
// Signature
public interface AuthCredentialStoreProvider {
```

## Architecture & Concepts
The AuthCredentialStoreProvider interface defines a factory contract for creating instances of IAuthCredentialStore. Its primary architectural role is to abstract the underlying storage mechanism for server authentication credentials, enabling a pluggable and configurable persistence strategy.

The most critical component of this design is the static CODEC field, a BuilderCodecMapCodec. This codec facilitates a polymorphic serialization system where different provider implementations (e.g., for file-based, in-memory, or database-backed stores) can be registered. During server initialization, configuration data specifying the desired store "Type" is deserialized via this codec, which in turn instantiates the corresponding provider implementation.

This pattern decouples the server's core authentication service from the concrete details of how and where credentials are stored, making the system highly modular and adaptable to different deployment environments.

## Lifecycle & Ownership
As an interface, AuthCredentialStoreProvider does not have a lifecycle itself. The following pertains to its concrete implementations.

-   **Creation:** An implementation of this provider is instantiated by the server's configuration loading system at startup. The system uses the `CODEC` to look up and construct the correct provider based on server configuration files.
-   **Scope:** A single provider instance is created and typically persists for the entire server session. It is held by the central authentication service.
-   **Destruction:** The provider instance is eligible for garbage collection during server shutdown when the services holding a reference to it are destroyed.

## Internal State & Concurrency
-   **State:** The interface itself is stateless. Implementations are expected to be effectively immutable after construction, holding only configuration data (e.g., a file path or database URL) required to create the IAuthCredentialStore.
-   **Thread Safety:** The interface contract does not enforce thread safety, but implementations of the `createStore` method should be thread-safe. It is expected that this method may be called to provision stores for different services, although typical usage involves a single call at initialization. The returned IAuthCredentialStore, however, must be thread-safe as it will handle concurrent authentication requests.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| createStore() | IAuthCredentialStore | Implementation-dependent | Factory method that constructs and returns a concrete IAuthCredentialStore. This may involve I/O operations like reading a file or establishing a database connection. Throws exceptions on configuration or I/O failure. |

## Integration Patterns

### Standard Usage
The provider is retrieved from a central service or configuration manager and used to create the credential store, which is then passed to the primary authentication service.

```java
// Pseudocode for server initialization
AuthCredentialStoreProvider provider = config.get("auth.provider"); // Deserialized via CODEC
IAuthCredentialStore credentialStore = provider.createStore();
AuthService authService = new AuthService(credentialStore);
```

### Anti-Patterns (Do NOT do this)
-   **Manual Instantiation:** Do not manually instantiate provider implementations (e.g., `new FileBasedProvider()`). This bypasses the server's configuration system and creates a rigid, non-configurable architecture. Always rely on the `CODEC` and the configuration loader.
-   **Multiple `createStore` Calls:** Avoid calling `createStore` multiple times within the same service. While the provider may allow it, this can lead to resource leaks (e.g., multiple database connection pools) or state conflicts (e.g., multiple instances attempting to lock the same file). The store should be created once and shared.

## Data Pipeline
This component acts as a factory in the control flow of server initialization, not as a processor in a data pipeline. Its output is a critical dependency for the authentication system.

> Flow:
> Server Configuration (YAML/JSON) -> Codec Deserializer -> **AuthCredentialStoreProvider Instance** -> `createStore()` -> IAuthCredentialStore Instance -> Injected into AuthService

