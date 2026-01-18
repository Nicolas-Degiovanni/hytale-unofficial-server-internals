---
description: Architectural reference for FloodFillEntryPoolProviderSimple
---

# FloodFillEntryPoolProviderSimple

**Package:** com.hypixel.hytale.server.spawning.util
**Type:** Transient

## Definition
```java
// Signature
public class FloodFillEntryPoolProviderSimple implements Resource<EntityStore> {
```

## Architecture & Concepts
The FloodFillEntryPoolProviderSimple class serves as a managed container for an object pool used in server-side entity spawning calculations. Its primary function is to provide an instance of FloodFillEntryPoolSimple, which is used to efficiently allocate and reuse objects during flood-fill algorithms. These algorithms are critical for determining valid spawn locations for entities across the world.

This class implements the Resource interface, tying its lifecycle directly to an EntityStore. This design pattern ensures that each distinct world storage unit (an EntityStore) receives its own isolated object pool. This encapsulation is critical for preventing state corruption and race conditions between different regions of the world that may be processed concurrently.

In essence, this class is a factory and a lifecycle bridge. It connects the engine's generic Resource management system to the specific needs of the spawning subsystem, ensuring that pooling resources are provisioned and managed consistently alongside the world data they service.

### Lifecycle & Ownership
- **Creation:** Instances are not created directly via a constructor in application code. The engine's resource management system creates instances by invoking the clone method when an EntityStore is initialized and requires this resource type.
- **Scope:** The object's lifetime is strictly scoped to the parent EntityStore that owns it. It persists as long as the EntityStore is loaded in memory.
- **Destruction:** The object is marked for garbage collection when its parent EntityStore is unloaded. There are no explicit de-initialization or cleanup methods; memory reclamation is handled by the JVM.

## Internal State & Concurrency
- **State:** This class is stateful. It holds a final reference to a FloodFillEntryPoolSimple instance. While the reference itself is immutable, the underlying pool object is highly mutable, as its internal state changes whenever objects are acquired or released.
- **Thread Safety:** This class is **not thread-safe**. It provides no internal locking mechanisms. The contained FloodFillEntryPoolSimple is also assumed to be non-reentrant. All access to the pool obtained from this provider **must** be synchronized externally by the calling system, which is typically the single thread responsible for processing the parent EntityStore.

**WARNING:** Unsynchronized, concurrent access to the pool provided by this class will lead to pool corruption, data races, and unpredictable server behavior.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getResourceType() | static ResourceType | O(1) | Retrieves the unique type identifier used by the resource system to manage this provider. |
| getPool() | FloodFillEntryPoolSimple | O(1) | Returns the underlying object pool instance. This is the primary entry point for consumers. |
| clone() | Resource | O(1) | Creates a new, distinct instance of the provider. Used by the resource system for provisioning. |

## Integration Patterns

### Standard Usage
This provider is intended to be accessed via the resource system of an existing EntityStore. The consumer is typically a higher-level spawning service that requires a pool for its calculations.

```java
// Correctly retrieve the provider from an EntityStore instance
EntityStore worldStore = ...;
FloodFillEntryPoolProviderSimple provider = worldStore.getResource(FloodFillEntryPoolProviderSimple.getResourceType());

// Access the pool to perform work
if (provider != null) {
    FloodFillEntryPoolSimple pool = provider.getPool();
    // Use the pool for spawning algorithms...
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never create an instance using `new FloodFillEntryPoolProviderSimple()`. This bypasses the engine's resource management system, resulting in an unmanaged object that will not be correctly associated with any EntityStore.
- **Pool Sharing:** Do not retrieve a provider from one EntityStore and use its pool for calculations related to a different EntityStore. This breaks the fundamental design principle of resource isolation and will cause severe state corruption.

## Data Pipeline
This class does not participate in a data transformation pipeline. Instead, it acts as a source of resources for control-flow operations within the spawning system.

> Flow:
> EntityStore Initialization -> Resource System calls **clone()** -> **FloodFillEntryPoolProviderSimple** is created and attached -> Spawning Algorithm requests Resource from EntityStore -> Algorithm calls **getPool()** -> Pool is used for flood-fill calculations

