---
description: Architectural reference for LegacyEntityTrackerSystems
---

# LegacyEntityTrackerSystems

**Package:** com.hypixel.hytale.server.core.modules.entity.tracker
**Type:** Utility / Container

## Definition
```java
// Signature
public class LegacyEntityTrackerSystems {
    // Contains only static methods and nested static system classes.
    // This class is not intended to be instantiated.
}
```

## Architecture & Concepts

The LegacyEntityTrackerSystems class is a container for a collection of specialized systems within the server's Entity Component System (ECS) framework. Its primary role is to manage the network synchronization of component data for entities that adhere to older, or "legacy", definitions. It acts as a bridge, ensuring that changes to specific components on the server are detected, packaged into network updates, and queued for transmission to relevant clients (viewers).

This class is a critical part of the **Entity Tracking Pipeline**. It does not perform the initial visibility calculations (determining *which* entities a player can see). Instead, its systems operate on the *results* of that calculation, which are stored in the EntityTrackerSystems.Visible component. For each entity deemed visible to a player, these systems check for state changes in specific components (e.g., Model, Skin, Equipment) and generate the corresponding ComponentUpdate packets.

The class is composed of two deprecated static utility methods and several nested static classes, each an implementation of EntityTickingSystem:

*   **LegacyEntityModel:** Tracks changes to an entity's ModelComponent and EntityScaleComponent.
*   **LegacyEntitySkin:** Tracks changes to a player's PlayerSkinComponent.
*   **LegacyEquipment:** Tracks changes to a LivingEntity's inventory and equipment.
*   **LegacyHideFromEntity:** A culling system that hides certain entities from a player's view based on their PlayerSettings, even if they are within the visible set.
*   **LegacyLODCull:** A Level-of-Detail (LOD) culling system that hides distant entities if their on-screen size would be too small to be meaningful, optimizing network and client-side performance.

These systems are designed to be scheduled and executed by the main server game loop's system runner. Their execution order is strictly managed via SystemGroup and Dependency definitions to prevent race conditions and ensure data consistency.

### Lifecycle & Ownership
- **Creation:** As a utility class, LegacyEntityTrackerSystems is never instantiated. Its nested system classes (e.g., LegacyEntityModel) are instantiated once by the ECS framework during server bootstrap, typically as part of the EntityModule setup.
- **Scope:** The nested systems are singletons within the ECS context and persist for the entire server session.
- **Destruction:** The systems are destroyed when the server shuts down and the ECS world is torn down.

## Internal State & Concurrency
- **State:** The container class itself is stateless. The nested system classes are stateful only in that they hold immutable references to ComponentType and Query objects defined at construction. They do not maintain mutable state across ticks.
- **Thread Safety:** The systems are designed for parallel execution. The `isParallel` method allows the ECS scheduler to run the `tick` method for different entity chunks concurrently on multiple threads. Thread safety is achieved by reading from the immutable Store and queueing state changes via the CommandBuffer or by calling thread-safe methods on components like EntityViewer. Direct mutation of shared state is strictly avoided.

## API Surface

The primary API is the collection of nested system classes, which are not called directly but are registered with the ECS. The static methods are deprecated and should not be used in new code.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| sendPlayerSelf(ref, store) | static void | O(N) | **Deprecated.** Manually constructs and sends a full entity update packet for a player to themselves. Bypasses the standard update-queuing mechanism. |
| clear(player, holder) | static boolean | O(1) | **Deprecated.** Clears the set of entities known to be tracked by a player. Prone to race conditions if not called from the world's main thread. |

## Integration Patterns

### Standard Usage
The systems within this class are not intended for direct invocation. They are automatically registered with the server's ECS SystemGroup during module initialization. Their logic is executed by the engine each tick as part of the entity tracking phase. A developer interacts with these systems implicitly by modifying the components they track.

```java
// An AI or game logic system modifies a component
ModelComponent model = store.getComponent(entityRef, ModelComponent.getComponentType());
model.setModel(newModel); // This internally flags the component as network-outdated

// On the next tick, LegacyEntityModel will automatically detect the change
// and queue a ComponentUpdate for all nearby players.
```

### Anti-Patterns (Do NOT do this)
- **Direct Invocation:** Do not call the static methods `sendPlayerSelf` or `clear`. They are deprecated and can disrupt the standard, managed flow of the entity tracking pipeline, leading to desynchronization.
- **System Instantiation:** Do not create instances of the nested system classes (e.g., `new LegacyEntityModel(...)`). The ECS framework is solely responsible for their lifecycle. Manual instantiation will result in a system that is not registered and will never be ticked.
- **Dependency Modification:** Altering the SystemGroup or Dependencies for these systems can break the entity tracking pipeline. For example, running `LegacyHideFromEntity` *before* `EntityTrackerSystems.CollectVisible` would cause it to operate on stale data from the previous tick.

## Data Pipeline
These systems function as processors in the entity-to-network data pipeline. They transform component state changes into discrete network messages.

> Flow:
> 1. Game Logic (e.g., AI, Player Input) modifies a component (e.g., ModelComponent) on an entity.
> 2. The component's internal `networkOutdated` flag is set to true.
> 3. **LegacyEntityTrackerSystems** (e.g., LegacyEntityModel) runs its query, finding all entities with a modified ModelComponent that are also visible to at least one player.
> 4. The system's `tick` method creates a `ComponentUpdate` packet containing the new model data.
> 5. This update is queued onto the `EntityViewer` component of each relevant player via `viewer.queueUpdate()`.
> 6. A final system (`EntityTrackerSystems.SendUpdates`) later collects all queued updates from the `EntityViewer` component, batches them into an `EntityUpdates` packet, and sends it over the network.

