---
description: Architectural reference for IResourceStorageProvider
---

# IResourceStorageProvider

**Package:** com.hypixel.hytale.server.core.universe.world.storage.resources
**Type:** Service Provider Interface (SPI)

## Definition
```java
// Signature
public interface IResourceStorageProvider {
```

## Architecture & Concepts
The IResourceStorageProvider interface defines a factory contract for creating or retrieving world-specific resource storage backends. It serves as a critical abstraction layer, decoupling the game's `World` object from the concrete implementation of its `IResourceStorage`. This allows different storage strategies (e.g., in-memory, file-based, database) to be plugged into the engine without modifying core world logic.

The static `CODEC` field is the central architectural element. It utilizes a `BuilderCodecMapCodec`, a Hytale-specific serialization system. This indicates that the concrete implementation of this interface is chosen and configured declaratively, typically within a world's configuration files. The server loads a world, reads its configuration, and uses this codec to deserialize the configuration into a specific IResourceStorageProvider instance. This pattern enables extensive moddability and configuration of world data persistence.

The generic parameter `<T extends WorldProvider>` on the `getResourceStorage` method is not directly used by the method's arguments but provides a mechanism for implementing classes to add further type constraints if necessary.

## Lifecycle & Ownership
As an interface, IResourceStorageProvider itself has no lifecycle. The following pertains to the objects that *implement* this interface.

-   **Creation:** Instances are not created directly via a constructor. They are instantiated by the Hytale codec system during server or world bootstrap, driven by configuration data. The `WorldProvider` is the likely orchestrator of this process.
-   **Scope:** An instance of an IResourceStorageProvider is scoped to a single `World`. It persists for the entire duration that its associated world is loaded in memory.
-   **Destruction:** The provider object is eligible for garbage collection when its parent `World` is unloaded and all references to it are released.

## Internal State & Concurrency
-   **State:** This interface is stateless. However, concrete implementations are expected to be highly stateful. They may manage file handles, database connections, or in-memory caches of resource data. The state is intrinsically tied to the `World` instance passed into its methods.
-   **Thread Safety:** The interface provides no thread-safety guarantees. Implementations may or may not be thread-safe. Access to a provider and the `IResourceStorage` it returns must be synchronized according to the rules of the world's threading model, typically confined to the main world thread.

**WARNING:** Never assume an implementation is thread-safe. Concurrent access from multiple threads without external synchronization will lead to data corruption and unpredictable behavior.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getResourceStorage(World) | IResourceStorage | O(1) Amortized | Retrieves the resource storage backend for the specified world. May involve lazy initialization on first call. |

## Integration Patterns

### Standard Usage
The provider is typically held as a field within a `World` or `WorldProvider` object. It is used to retrieve the storage interface, which is then used for all subsequent resource operations for that world.

```java
// Within a World or WorldProvider context
// this.storageProvider is deserialized from world config

IResourceStorage storage = this.storageProvider.getResourceStorage(this.world);
Optional<Resource> myResource = storage.getResource("hytale:my_item");
```

### Anti-Patterns (Do NOT do this)
-   **Cross-World Usage:** Do not use a provider instance from one world to get storage for a different world. The provider is intrinsically linked to the world it was created for.
-   **Type Casting:** Avoid casting an IResourceStorageProvider to a concrete implementation. This violates the abstraction and couples game logic to a specific storage strategy, defeating the purpose of the interface.
-   **External Caching:** Do not cache the `IResourceStorage` object returned by `getResourceStorage` in a global or static context. Its lifecycle is managed by the `World` and it may become invalid when the world unloads.

## Data Pipeline
This component acts as a factory in a control-flow pipeline, not a data-processing pipeline.

> Flow:
> World Configuration File -> Server Bootstrap -> **IResourceStorageProvider (Deserialized)** -> `World.getResourceStorage()` call -> `IResourceStorage` instance returned

