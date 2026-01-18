---
description: Architectural reference for ComponentRegistry
---

# ComponentRegistry

**Package:** com.hypixel.hytale.component
**Type:** Singleton

## Definition
```java
// Signature
public class ComponentRegistry<ECS_TYPE> implements IComponentRegistry<ECS_TYPE> {
```

## Architecture & Concepts

The ComponentRegistry is the central nervous system of Hytale's Entity Component System (ECS). It is the authoritative source for the definition, registration, and lifecycle management of all ECS primitives: Components, Systems, Resources, and Event Types. Its primary architectural role is to act as a dynamic schema manager for one or more ECS worlds, known as Stores.

The core design revolves around an immutable snapshot pattern, implemented via the inner class ComponentRegistry.Data. Any modification to the registry's schema—such as registering a new component or system—does not alter the live state directly. Instead, it triggers the creation of a new, versioned, immutable Data snapshot. This snapshot is then atomically distributed to all active Store instances. This architecture ensures that each Store operates on a consistent and thread-safe view of the ECS schema, even as the registry is being modified by other threads.

This class is responsible for:
*   **Type Registration:** Maintaining canonical lists of all available ComponentTypes, ResourceTypes, and SystemTypes. It assigns unique, recyclable integer indices to each registered item for high-performance lookups.
*   **System Ordering:** Calculating the correct execution order for all registered systems based on their declared dependencies. This produces a sorted list of systems that is part of the Data snapshot.
*   **Store Management:** Managing the lifecycle of Store objects, which are the concrete execution environments for the ECS. It orchestrates the propagation of schema updates to these Stores.
*   **Serialization:** Providing the master Codec for serializing and deserializing entities (Holders) to and from BSON format. This is achieved by dynamically building a map-based codec from all registered, serializable components.

The generic parameter ECS_TYPE serves as a context handle, allowing the registry and its associated objects to be tied to a specific environment (e.g., a client world vs. a server world) without explicit parameter passing.

## Lifecycle & Ownership

- **Creation:** The ComponentRegistry is a foundational service, instantiated once at the beginning of the application lifecycle (e.g., during client or server startup). It is designed to be a long-lived singleton.
- **Scope:** It persists for the entire application session. Its state represents the complete and current definition of the game's ECS capabilities.
- **Destruction:** The shutdown method must be called during application termination. This initiates a graceful shutdown sequence, interrupting its internal reference-cleaning thread and explicitly shutting down all managed Store instances to prevent resource leaks. A shutdown registry will reject all subsequent registration attempts.

A dedicated daemon thread, HolderReferenceThread, runs for the lifetime of the registry. It uses a ReferenceQueue to monitor weakly-referenced Holder objects, ensuring that entity wrappers that are no longer strongly referenced can be garbage collected and removed from internal tracking sets.

## Internal State & Concurrency

The ComponentRegistry is a highly stateful and mutable class designed for concurrent access. Its thread safety is managed through a sophisticated multi-lock strategy.

- **State:** The primary state consists of several dynamically-sized arrays and maps that store registration data for components, resources, and systems. This includes their string IDs, class types, suppliers, and codecs. The most critical piece of state is the `data` field, which holds the current immutable ComponentRegistry.Data snapshot.

- **Thread Safety:** This class is thread-safe, but access patterns are critical.
    - **dataLock (StampedLock):** Protects the raw registration arrays (`componentIds`, `systems`, etc.). Write operations (register, unregister) acquire a full write lock. This lock is used to ensure atomicity when modifying the fundamental schema definitions.
    - **storeLock (StampedLock):** Protects the `stores` array. This lock is acquired when adding or removing a Store from the registry.
    - **dataUpdateLock (ReentrantReadWriteLock):** Protects the `data` field itself. After a new Data snapshot is created, this write lock is acquired to atomically swap the reference, ensuring no thread can access the `data` field while it is being updated.
    - **The Snapshot Pattern:** The use of the immutable ComponentRegistry.Data object is the key to its concurrency model. Systems running inside a Store can read from their local copy of the Data snapshot without any locking, as its contents are guaranteed not to change. When a registration change occurs, a new Data object is created, and the reference is swapped. This copy-on-write approach minimizes lock contention and provides excellent read performance for active game loops.

**WARNING:** Direct modification of the internal arrays would be catastrophic, bypassing all locking and snapshotting mechanisms and leading to undefined behavior.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| registerComponent(...) | ComponentType | O(N) | Registers a new component definition. Acquires a write lock. Throws if ID is duplicated. |
| registerResource(...) | ResourceType | O(N) | Registers a new global resource definition. Acquires a write lock. |
| registerSystem(...) | void | O(N^2) | Registers a new system. This is a heavy operation that triggers a recalculation of the system execution order. |
| unregisterComponent(...) | void | O(N) | Removes a component definition. Also unregisters any systems that depend on it. |
| addStore(...) | Store | O(1) | Creates and registers a new ECS world (Store). Acquires a write lock on the store list. |
| removeStore(...) | void | O(N) | Shuts down and removes a Store. |
| newHolder(...) | Holder | O(1) | Creates a new entity wrapper (Holder) associated with this registry. |
| serialize(Holder) | BsonDocument | O(C) | Encodes an entity and its components into a BSON document using the current data snapshot. C is component count. |
| deserialize(BsonDocument) | Holder | O(C) | Decodes a BSON document into a new entity Holder. |

## Integration Patterns

### Standard Usage

The registry should be treated as a singleton service obtained from a central context or dependency injection container. Engine modules register their components and systems during an initialization phase.

```java
// During engine initialization
ComponentRegistry registry = EcsContext.getRegistry();

// Register a component with a string ID for serialization
registry.registerComponent(Position.class, "hytale:position", POSITION_CODEC);

// Register a transient component without serialization
registry.registerComponent(Velocity.class, Velocity::new);

// Register a system
registry.registerSystem(new PhysicsSystem());

// After all registrations, create a world
Store world = registry.addStore(worldContext, resourceStorage);
```

### Anti-Patterns (Do NOT do this)

- **Dynamic Registration During Gameplay:** Do not register or unregister components or systems in the middle of a game tick. These are heavyweight operations that cause global lock contention and force a full data propagation to all Stores. All registrations should occur during well-defined loading phases.
- **Holding Stale Data References:** Do not cache the result of `registry.getData()` for long periods. While the returned Data object is immutable, the registry may have produced a newer version. Always fetch the latest data from the Store you are operating on.
- **Ignoring Shutdown:** Failure to call `shutdown()` on the registry will result in resource leaks, as the internal threads and managed Stores will not be properly terminated.

## Data Pipeline

The ComponentRegistry is not part of a real-time data processing pipeline; rather, it *defines* the structure that such pipelines (within Stores) will use. Its primary data flow relates to schema updates.

> Flow:
> `registerSystem(MySystem)` -> **ComponentRegistry** (acquires `dataLock`) -> Internal system arrays are modified -> `updateData0` is called -> New `ComponentRegistry.Data` snapshot is created -> **ComponentRegistry** (acquires `dataUpdateLock` and `storeLock`) -> `this.data` reference is swapped -> All active `Store` instances are notified with the new `Data` snapshot.

