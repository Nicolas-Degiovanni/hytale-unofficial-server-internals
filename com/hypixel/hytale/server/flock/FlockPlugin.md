---
description: Architectural reference for FlockPlugin
---

# FlockPlugin

**Package:** com.hypixel.hytale.server.flock
**Type:** Singleton

## Definition
```java
// Signature
public class FlockPlugin extends JavaPlugin {
```

## Architecture & Concepts
The FlockPlugin is the central authority and initialization point for the server-side NPC flocking system. As a subclass of JavaPlugin, it is automatically discovered and managed by the server's plugin lifecycle manager. Its primary architectural role is to bootstrap the entire flocking feature set by integrating it into the server's core Entity Component System (ECS) and asset management pipelines.

This class acts as a service registry and factory for all flock-related constructs. During the server's `setup` phase, it performs several critical functions:
1.  **Asset Registration:** It registers the FlockAsset, making flock configuration data loadable from game files.
2.  **Component Registration:** It defines and registers the core ECS components that represent flocking data: Flock, FlockMembership, and PersistentFlockData. These components are the canonical data stores for group state.
3.  **System Registration:** It registers numerous systems (e.g., FlockSystems, FlockMembershipSystems) that contain the logic for flock behavior. These systems operate on entities possessing the aforementioned components during the main game loop.
4.  **NPC Behavior Integration:** It extends the NPC decision-making engine (NPCPlugin) by registering new core component types, such as JoinFlock and LeaveFlock actions, and conditions like FlockSizeCondition. This allows content designers to create complex flocking behaviors within NPC state machines.
5.  **Service Provision:** It exposes a static singleton instance and a suite of static utility methods (e.g., trySpawnFlock, createFlock) that provide a high-level, simplified API for other game systems to create and manage flocks without needing to interact directly with the low-level ECS details.

In essence, FlockPlugin is the glue that binds data (assets, components) and logic (systems, NPC behaviors) into a cohesive, engine-integrated feature.

### Lifecycle & Ownership
-   **Creation:** A single instance of FlockPlugin is instantiated by the server's plugin loader during the server bootstrap sequence. The constructor requires a JavaPluginInit context, which is provided by the loader.
-   **Scope:** The instance is a global singleton that persists for the entire lifetime of the server process. The static `instance` field is assigned in the `setup` method, making it accessible via the static `get()` method thereafter.
-   **Destruction:** The `shutdown` method is invoked by the plugin loader when the server is shutting down. This provides a hook for releasing resources, though the current implementation is empty.

## Internal State & Concurrency
-   **State:** The FlockPlugin maintains a minimal but critical internal state.
    -   **Component Types:** It holds immutable references to the registered `ComponentType` objects for Flock, FlockMembership, and PersistentFlockData. These act as typed keys for all future ECS interactions.
    -   **Prefab Remappings:** The `prefabFlockRemappings` field is a mutable, concurrent map. It is designed to solve a specific data integrity problem: when a prefab containing a flock is pasted into the world, the flock must be treated as a new, unique entity, not a reference to the original. This map stores temporary UUID remappings for the duration of a paste operation, managed by the internal PrefabPasteEventSystem.

-   **Thread Safety:** This class is designed to be thread-safe under specific conditions.
    -   Initialization via `setup` is guaranteed by the engine to occur in a single-threaded context during startup.
    -   The `prefabFlockRemappings` map uses Int2ObjectConcurrentHashMap, making its access safe from any thread.
    -   **WARNING:** The static utility methods like `trySpawnFlock` and `createFlock` operate directly on the ECS `Store`. The caller is responsible for ensuring these methods are invoked from a context that has safe access to the ECS, typically the main server thread or via a command buffer. Calling these methods from an arbitrary asynchronous thread without proper synchronization will lead to race conditions and world corruption.

## API Surface
The public API consists primarily of static utility methods that serve as the main entry points for interacting with the flocking system.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| get() | FlockPlugin | O(1) | Returns the global singleton instance of the plugin. |
| setup() | void | O(N) | **Engine Internal.** Registers all flock-related assets, components, and systems. |
| getFlockComponentType() | ComponentType | O(1) | Returns the component type handle for the Flock component. |
| getFlockMembershipComponentType() | ComponentType | O(1) | Returns the component type handle for the FlockMembership component. |
| trySpawnFlock(...) | Ref | O(N) | High-level utility to spawn a full flock of N members around a leader. |
| createFlock(...) | Ref | O(1) | Low-level utility to create a single, empty flock entity. |
| getFlockReference(...) | Ref | O(1) | Retrieves the flock entity reference for a given member entity. |
| isFlockMember(...) | boolean | O(1) | Checks if an entity is part of any flock. |

## Integration Patterns

### Standard Usage
The most common use case is spawning a flock of NPCs. This is accomplished by calling the static `trySpawnFlock` method, typically during world generation or in response to a game event. This abstracts away the complexity of creating the flock entity, spawning each member, and linking them together.

```java
// Example: Spawning a flock for an existing NPC leader
// Assume npcRef, npc, and store are valid and initialized.

Vector3d spawnPosition = npc.getPosition();
int desiredFlockSize = 5; // Spawn 4 additional members

// This single call creates the flock entity, spawns members,
// and correctly assigns FlockMembership components.
FlockPlugin.trySpawnFlock(
    npcRef,
    npc,
    store,
    npc.getRoleIndex(),
    spawnPosition,
    null, // Use default rotation
    desiredFlockSize,
    (spawnedNpc, spawnedRef, spawnedStore) -> {
        // Optional: Post-spawn logic for each new member
    }
);
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Do not call `new FlockPlugin()`. The server's plugin loader is solely responsible for its lifecycle. Use the static `FlockPlugin.get()` method to access the instance.
-   **Premature Access:** Do not call `FlockPlugin.get()` before the server's plugin setup phase is complete. The static instance will be null, resulting in a NullPointerException.
-   **Unsafe ECS Mutation:** Do not call static methods like `trySpawnFlock` or `createFlock` from asynchronous threads without dispatching the operation to the main world thread. Direct mutation of the ECS `Store` from other threads is not safe and will cause unpredictable behavior.

## Data Pipeline
The FlockPlugin itself is not part of a data processing pipeline; rather, it sets up the components and systems that are. The data flow for flock creation, a primary function it enables, is as follows:

> Flow:
> External System Call (e.g., World Generator) -> **FlockPlugin.trySpawnFlock()** -> **FlockPlugin.createFlock()** -> ECS `Store.addEntity()` (for Flock entity) -> NPCPlugin.spawnEntity() [Loop] -> ECS `Store.addEntity()` (for member NPCs) -> FlockMembershipSystems.join() -> ECS `Store.addComponent()` (adds FlockMembership to link member to flock)

