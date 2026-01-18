---
description: Architectural reference for SortBufferProviderResource
---

# SortBufferProviderResource

**Package:** com.hypixel.hytale.server.npc.components
**Type:** Transient

## Definition
```java
// Signature
public class SortBufferProviderResource implements Resource<EntityStore> {
```

## Architecture & Concepts
The SortBufferProviderResource is a specialized component that acts as a container for a performance-critical utility, the BucketList.SortBufferProvider. In Hytale's server architecture, a Resource is a piece of data or functionality attached to a larger engine object. This specific resource is designed to be attached to an EntityStore.

Its primary function is to mitigate memory allocation overhead during the server's NPC processing tick. Many spatial partitioning and entity querying algorithms, which likely use the BucketList data structure, require frequent sorting operations. By providing a reusable buffer via SortBufferProvider, this resource ensures that these operations do not generate excessive garbage, leading to more stable server performance.

This class serves as the bridge between the EntityStore's lifecycle and the NPC system's performance requirements. Each EntityStore instance receives its own dedicated SortBufferProvider, ensuring that sorting operations within one store do not interfere with another.

### Lifecycle & Ownership
- **Creation:** Instances are not created directly. The resource's type is registered with the engine by the NPCPlugin. When a new EntityStore is initialized, the engine's resource management system identifies the need for this resource and creates a new instance, likely by invoking the clone method on a registered prototype.
- **Scope:** The lifecycle of a SortBufferProviderResource is tightly bound to its host EntityStore. It exists only as long as the EntityStore is active in the world.
- **Destruction:** The resource is eligible for garbage collection immediately after its parent EntityStore is unloaded or destroyed. It holds no native resources and requires no explicit cleanup.

## Internal State & Concurrency
- **State:** The SortBufferProviderResource itself is a stateless wrapper. However, the internal BucketList.SortBufferProvider object it holds is highly **mutable**. Its purpose is to cache and reuse large array buffers for sorting algorithms.
- **Thread Safety:** This component is **not thread-safe**. It is designed for use exclusively by the single thread responsible for updating its parent EntityStore. Concurrent access to the underlying SortBufferProvider from multiple threads will result in buffer corruption, race conditions, and catastrophic server instability. All interactions must be synchronized externally or confined to the main server tick.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getResourceType() | static ResourceType | O(1) | Retrieves the globally registered type definition for this resource. |
| getSortBufferProvider() | BucketList.SortBufferProvider | O(1) | Returns the stateful, mutable buffer provider instance. |
| clone() | Resource | O(1) | Creates a new, distinct instance of the resource with its own SortBufferProvider. |

## Integration Patterns

### Standard Usage
The resource is intended to be retrieved from an EntityStore within a system that performs entity processing. The provider is then passed to sorting-dependent algorithms.

```java
// Within a server system that operates on an EntityStore
EntityStore store = ...; // Obtain the target EntityStore

// Retrieve the resource using its registered type
SortBufferProviderResource resource = store.getResource(SortBufferProviderResource.getResourceType());

if (resource != null) {
    BucketList.SortBufferProvider provider = resource.getSortBufferProvider();
    // Pass the provider to a system that requires it for sorting
    someNpcSystem.updateEntities(store.getEntities(), provider);
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never call `new SortBufferProviderResource()`. The engine's resource system is solely responsible for managing the lifecycle of this object. Direct creation bypasses its attachment to an EntityStore, rendering it useless.
- **Cross-Store Usage:** Do not retrieve a SortBufferProvider from one EntityStore and use it for operations on another. Each provider is scoped to its host, and sharing them can lead to unpredictable sorting results and state corruption.
- **Caching the Provider:** Do not cache the SortBufferProvider instance in a long-lived service. Always retrieve the resource from the EntityStore at the beginning of an operation, as the store and its associated resources may be unloaded at any time.

## Data Pipeline
This component does not process a stream of data. Instead, it provides a critical utility that facilitates a data processing step within the NPC update loop.

> Flow:
> EntityStore Update Tick -> NPC System Queries Store -> **SortBufferProviderResource** is retrieved -> SortBufferProvider is extracted -> Provider is passed to BucketList.sort -> Entities are sorted in-place using recycled buffers.

