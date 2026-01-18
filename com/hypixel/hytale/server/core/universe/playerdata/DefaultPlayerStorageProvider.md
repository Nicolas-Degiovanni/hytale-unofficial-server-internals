---
description: Architectural reference for DefaultPlayerStorageProvider
---

# DefaultPlayerStorageProvider

**Package:** com.hypixel.hytale.server.core.universe.playerdata
**Type:** Singleton

## Definition
```java
// Signature
public class DefaultPlayerStorageProvider implements PlayerStorageProvider {
```

## Architecture & Concepts
The DefaultPlayerStorageProvider is a concrete, stateless implementation of the PlayerStorageProvider strategy interface. It functions as a Singleton delegator, providing a default, out-of-the-box mechanism for player data persistence.

Its primary architectural role is to decouple the server universe from a specific storage implementation. By default, it hardcodes the storage strategy to use the DiskPlayerStorageProvider, ensuring that a standard file-system-based persistence layer is always available if no alternative is configured. This class embodies the "convention over configuration" principle for player data, providing a sensible default that works for standalone servers without requiring explicit setup.

It is not a storage provider in its own right; it contains no logic for reading or writing data. Instead, it acts as a static pointer to the canonical disk-based provider.

## Lifecycle & Ownership
- **Creation:** The single `INSTANCE` is created and initialized by the JVM during class loading. Its lifecycle is not managed by any dependency injection framework or factory.
- **Scope:** Application-wide. The Singleton instance persists for the entire lifetime of the server process.
- **Destruction:** The instance is eligible for garbage collection only upon JVM shutdown. No explicit cleanup is required.

## Internal State & Concurrency
- **State:** This class is stateless and immutable. It holds no instance fields and its static fields are final. Its behavior is constant and predictable.
- **Thread Safety:** Inherently thread-safe. As a stateless Singleton, it can be accessed and used by any number of threads concurrently without risk of race conditions or data corruption. No synchronization mechanisms are necessary.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getPlayerStorage() | PlayerStorage | O(1) | Returns the concrete PlayerStorage implementation. This call is a direct delegation to the default DiskPlayerStorageProvider. |

## Integration Patterns

### Standard Usage
This class is rarely interacted with directly. Instead, it is used by the server's configuration or universe bootstrap logic as a default fallback. A system requiring a PlayerStorageProvider would use this instance if no other provider was specified in its configuration.

```java
// Example: A server factory using the default provider
PlayerStorageProvider configuredProvider = serverConfig.getPlayerStorageProvider();

if (configuredProvider == null) {
    // If no provider is specified, fall back to the default disk-based system
    configuredProvider = DefaultPlayerStorageProvider.INSTANCE;
}

PlayerStorage storage = configuredProvider.getPlayerStorage();
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not call `new DefaultPlayerStorageProvider()`. This is wasteful and defeats the purpose of the Singleton pattern. Always use the static `DefaultPlayerStorageProvider.INSTANCE` field.
- **Conditional Logic:** Do not write code that checks `if (provider instanceof DefaultPlayerStorageProvider)`. The purpose of the interface is to abstract away the concrete implementation. Your code should be agnostic to the specific provider being used.

## Data Pipeline
This class does not process data. It acts as a factory at the beginning of the data pipeline, providing the component that will be responsible for persistence.

> Flow:
> Server Configuration -> **DefaultPlayerStorageProvider** (provides the storage engine) -> DiskPlayerStorage (the engine itself) -> Filesystem I/O

