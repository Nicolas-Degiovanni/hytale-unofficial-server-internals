---
description: Architectural reference for EmptyResourceStorageProvider
---

# EmptyResourceStorageProvider

**Package:** com.hypixel.hytale.server.core.universe.world.storage.resources
**Type:** Singleton

## Definition
```java
// Signature
public class EmptyResourceStorageProvider implements IResourceStorageProvider {
```

## Architecture & Concepts
The EmptyResourceStorageProvider is a specific implementation of the IResourceStorageProvider strategy interface. Its primary architectural function is to serve as a **Null Object**. It provides a valid, non-null IResourceStorage object that performs no operations.

This component is critical for worlds or dimensions that do not require persistent state or resources. By assigning this provider, the engine can bypass all disk I/O, serialization, and caching logic related to resource storage for a given world, resulting in a significant performance gain and reduced complexity. It allows the rest of the world management system to operate on a consistent IResourceStorageProvider interface without special-casing for worlds that lack storage.

The provider is identified and loaded via its static ID field, "Empty", which is used in world configuration files. Its CODEC ensures that any deserialization of this provider type resolves to the single, shared INSTANCE, enforcing the singleton pattern.

### Lifecycle & Ownership
- **Creation:** The singleton INSTANCE is created and initialized by the Java ClassLoader when the EmptyResourceStorageProvider class is first referenced. Its lifecycle is not managed by any dependency injection framework or service locator.
- **Scope:** Application-wide. A single instance exists for the entire lifetime of the server process.
- **Destruction:** The object is reclaimed by the JVM during server shutdown. No manual cleanup is required or possible.

## Internal State & Concurrency
- **State:** This class is **stateless and immutable**. It contains no instance fields and its behavior is constant.
- **Thread Safety:** This class is unconditionally thread-safe. Its stateless nature guarantees that it can be safely accessed and used by any number of threads concurrently without locks or synchronization primitives.

## API Surface
The public API consists solely of the contract defined by the IResourceStorageProvider interface.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getResourceStorage(World world) | IResourceStorage | O(1) | Returns the singleton instance of EmptyResourceStorage. This operation is instantaneous and has no side effects. |

## Integration Patterns

### Standard Usage
This provider is not intended to be used directly in procedural code. Instead, it is specified declaratively in world configuration data. The engine's world loader deserializes this configuration and associates the provider with the World object.

**WARNING:** The following code is for conceptual understanding only. Developers should never need to resolve this provider manually.

```java
// Concept: The engine resolves the provider based on world configuration
// WorldConfig.json -> "storageProvider": "Empty"

// During world initialization, the engine does something similar to this:
IResourceStorageProvider provider = providerRegistry.get("Empty"); // Resolves to EmptyResourceStorageProvider.INSTANCE
IResourceStorage storage = provider.getResourceStorage(currentWorld);

// The 'storage' object is now a no-op EmptyResourceStorage instance.
storage.saveResource(someResource); // This call does nothing.
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not use `new EmptyResourceStorageProvider()`. The class is a singleton and must only be accessed via its static `INSTANCE` field or through a registry that resolves to it.
- **Type Checking:** Avoid checking if a provider is an instance of this class. The purpose of the Null Object pattern is to eliminate such checks. Code should operate against the IResourceStorageProvider and IResourceStorage interfaces without knowledge of the concrete implementation.

   ```java
   // ANTI-PATTERN: Do not do this.
   if (provider instanceof EmptyResourceStorageProvider) {
       // Skip saving...
   } else {
       provider.getResourceStorage(world).save(data);
   }

   // CORRECT: Let the object handle the logic.
   // If the provider is an EmptyResourceStorageProvider, the save() call will correctly do nothing.
   provider.getResourceStorage(world).save(data);
   ```

## Data Pipeline
The EmptyResourceStorageProvider acts as a terminal point in the data pipeline, effectively short-circuiting any storage operations.

> Flow:
> World Configuration File -> Deserializer (using CODEC) -> **EmptyResourceStorageProvider.INSTANCE** -> World.getResourceStorage() -> EmptyResourceStorage -> (Operation Discarded)

