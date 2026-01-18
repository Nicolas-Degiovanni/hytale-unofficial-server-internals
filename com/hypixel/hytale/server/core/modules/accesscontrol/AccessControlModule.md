---
description: Architectural reference for AccessControlModule
---

# AccessControlModule

**Package:** com.hypixel.hytale.server.core.modules.accesscontrol
**Type:** Singleton

## Definition
```java
// Signature
public class AccessControlModule extends JavaPlugin {
```

## Architecture & Concepts
The AccessControlModule is a core server plugin responsible for managing and enforcing player access rules, primarily bans and whitelists. It functions as a central authority that determines whether a connecting player is allowed to join the server.

Architecturally, it serves two primary roles:
1.  **Event Interceptor:** It hooks into the player connection lifecycle by listening for the PlayerSetupConnectEvent. Upon firing, it interrogates its registered providers to build a consensus on whether the connection should be terminated.
2.  **Provider Registry:** It maintains a collection of AccessProvider implementations. This decouples the core connection logic from the specific methods of access control (e.g., file-based whitelist, database-driven ban list). The module ships with default providers for Hytale's native ban and whitelist files but is extensible for third-party plugins to register their own custom logic.

The system is designed for asynchronicity. The primary check, getDisconnectReason, returns a CompletableFuture, allowing individual AccessProviders to perform non-blocking operations like network requests or database queries without stalling the main server thread. The module then aggregates the results from all providers to make a final decision.

It also includes a deserialization system for different ban types (e.g., timed, infinite) via the BanParser interface, making the ban system itself extensible.

## Lifecycle & Ownership
-   **Creation:** Instantiated by the server's plugin loader during the server bootstrap sequence. The static singleton instance is set within the constructor, making it immediately available via the static get method.
-   **Scope:** The AccessControlModule is a session-scoped object. It persists for the entire runtime of the server.
-   **Destruction:** The shutdown method is invoked by the plugin loader during the server shutdown process. This is a critical phase where the module ensures that any in-memory changes to access lists (bans, whitelists) are persisted to disk via the syncSave methods of its providers.

## Internal State & Concurrency
-   **State:** The module's state is mutable. It maintains live collections of access providers and ban parsers. The default providers, HytaleWhitelistProvider and HytaleBanProvider, manage in-memory caches of the whitelist and ban lists, which are loaded from disk on startup.
-   **Thread Safety:** This class is designed to be thread-safe.
    -   The providerRegistry uses a CopyOnWriteArrayList. This is an explicit choice for concurrency, optimized for scenarios where reads (iterating providers during a connection check) are far more frequent than writes (registering a new provider). Iterations are performed on an immutable snapshot, preventing concurrent modification exceptions.
    -   The parsers map uses a ConcurrentHashMap, allowing for thread-safe registration and retrieval of ban parsers at any point in the server lifecycle.
    -   The core connection check logic is asynchronous, preventing I/O-bound AccessProviders from blocking the thread handling the player connection event.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| get() | AccessControlModule | O(1) | Retrieves the static singleton instance of the module. |
| registerBanParser(type, parser) | void | O(1) | Registers a parser for a custom ban type. Throws IllegalArgumentException if the type is already registered. |
| registerAccessProvider(provider) | void | O(N) | Registers a new AccessProvider. The complexity is due to the copy-on-write nature of the underlying list. |
| parseBan(type, object) | Ban | O(1) | Deserializes a JsonObject into a Ban object using a registered parser. Throws IllegalArgumentException if no parser exists for the type. |

## Integration Patterns

### Standard Usage
A third-party plugin would typically interact with this module to add a new form of access control, such as checking a global reputation service.

```java
// In your plugin's setup() method:

// 1. Get the singleton instance
AccessControlModule accessControl = AccessControlModule.get();

// 2. Create and register a custom provider
// This provider might check a player's UUID against a remote API
MyCustomApiProvider myProvider = new MyCustomApiProvider();
accessControl.registerAccessProvider(myProvider);
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Do not use `new AccessControlModule()`. The module is managed by the server's plugin system. Always use `AccessControlModule.get()` to retrieve the active instance.
-   **Blocking Providers:** When implementing a custom AccessProvider, do not perform blocking I/O operations directly within the getDisconnectReason method. Wrap the call in a CompletableFuture to align with the module's asynchronous design and avoid stalling the server.
-   **Ignoring Aggregation Logic:** The module processes providers in registration order and accepts the *first* non-empty disconnect reason. Be mindful of the order in which you register providers if multiple could deny the same player.

## Data Pipeline
The primary data flow occurs when a player attempts to connect to the server.

> Flow:
> Player Connection Attempt -> Server fires PlayerSetupConnectEvent -> **AccessControlModule** (listener) -> getDisconnectReason(uuid) -> All registered AccessProviders are queried in parallel -> CompletableFutures are combined -> The first non-empty reason is selected -> Event is cancelled with the reason -> Server sends disconnect packet to client.

