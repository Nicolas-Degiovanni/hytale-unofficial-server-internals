---
description: Architectural reference for DefaultResourceStorageProvider
---

# DefaultResourceStorageProvider

**Package:** com.hypixel.hytale.server.core.universe.world.storage.resources
**Type:** Singleton

## Definition
```java
// Signature
public class DefaultResourceStorageProvider implements IResourceStorageProvider {
```

## Architecture & Concepts
The DefaultResourceStorageProvider is a foundational component in the server's world data persistence layer. It serves as a static, default implementation of the IResourceStorageProvider interface.

Its primary architectural role is not to implement storage logic itself, but to act as a **stable delegation endpoint**. It forwards all requests for a resource storage interface to a concrete implementation, specifically the DiskResourceStorageProvider. This design decouples the core world management systems from the specifics of how world data is physically stored.

By providing a constant, serializable singleton (via its CODEC field), the engine ensures that a valid, disk-based storage provider is always available as a fallback during world deserialization or initial creation, preventing configuration errors from halting server startup. It embodies a form of the **Strategy Pattern**, where this class represents the default, non-configurable strategy for resource persistence.

## Lifecycle & Ownership
- **Creation:** The singleton INSTANCE is instantiated by the JVM class loader when the DefaultResourceStorageProvider class is first referenced. Its lifecycle is not managed by a dependency injection framework or any game-level factory.
- **Scope:** Application-wide. The singleton instance persists for the entire lifetime of the server process.
- **Destruction:** The object is eligible for garbage collection only upon JVM shutdown. No explicit cleanup methods are provided or required.

## Internal State & Concurrency
- **State:** This class is **stateless and immutable**. It contains no instance fields and its static fields are final. Its behavior is fixed at compile time.
- **Thread Safety:** The class is inherently **thread-safe**. As a stateless singleton, it can be accessed from any thread without synchronization. The responsibility for thread-safe I/O operations lies with the IResourceStorage instance it provides, not with the provider itself.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getResourceStorage(World world) | IResourceStorage | O(1) | Retrieves the default resource storage handler for a given world. This method delegates the request directly to the underlying DiskResourceStorageProvider. |

## Integration Patterns

### Standard Usage
This class is typically not invoked directly by game logic developers. It is primarily used by the server's world loading and configuration systems as a default value. The presence of a static CODEC indicates its role in data serialization frameworks.

When a world's configuration does not specify a custom resource storage provider, the system falls back to this default implementation.

```java
// Example of framework-level usage during world loading
IResourceStorageProvider provider = worldConfig.getStorageProvider().orElse(DefaultResourceStorageProvider.INSTANCE);
IResourceStorage storage = provider.getResourceStorage(world);
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not use `new DefaultResourceStorageProvider()`. The class is a singleton and should only be accessed via its static `INSTANCE` field to maintain a single point of reference.
- **Unnecessary Abstraction:** Do not write code that specifically checks for this type. Treat it as a generic IResourceStorageProvider. The purpose of the interface is to hide this specific implementation detail.

## Data Pipeline
This component acts as a factory or provider at the beginning of a data access pipeline. It does not process data itself but rather provides the object that will.

> Flow:
> World Loading System -> Configuration Check -> **DefaultResourceStorageProvider** (as fallback) -> Returns DiskResourceStorageProvider -> Filesystem Access

