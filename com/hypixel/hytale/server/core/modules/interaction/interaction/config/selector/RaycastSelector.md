---
description: Architectural reference for RaycastSelector
---

# RaycastSelector

**Package:** com.hypixel.hytale.server.core.modules.interaction.interaction.config.selector
**Type:** Configuration / Factory

## Definition
```java
// Signature
public class RaycastSelector extends SelectorType {
```

## Architecture & Concepts

The RaycastSelector is a data-driven configuration class that defines the parameters for a server-side line-of-sight targeting query. It is a fundamental component of the Interaction system, used to determine what an entity is looking at, which can be either a block or another entity.

Architecturally, this class embodies the separation of configuration from runtime logic.
*   **RaycastSelector (Outer Class):** This is the immutable configuration object. It is designed to be deserialized from asset files (e.g., JSON) using its static CODEC field. It holds parameters like raycast distance, origin offset, and filters for block types or fluids. It acts as a template or a blueprint for a specific type of raycast.
*   **RuntimeSelector (Inner Class):** This is a stateful, short-lived object created by the RaycastSelector. It performs the actual raycasting calculations during the game tick. Each instance maintains the results of its last query, making it specific to a single interaction context.

This two-part design allows game designers to define a wide variety of targeting behaviors in data files without modifying core engine code, promoting a highly modular and data-driven workflow.

### Lifecycle & Ownership
-   **Creation:** The outer RaycastSelector is instantiated by the asset loading system via its BuilderCodec. It is not intended for manual instantiation in game logic. The inner RuntimeSelector is created on-demand by calling the `newSelector` factory method on a configured RaycastSelector instance.
-   **Scope:** A RaycastSelector instance persists as long as its defining asset is loaded in memory. The RuntimeSelector instance has a much shorter lifecycle, typically scoped to a single, continuous interaction for one entity. It is created when an interaction begins and discarded when it ends.
-   **Destruction:** The RaycastSelector is eligible for garbage collection when the AssetRegistry unloads its parent asset. The RuntimeSelector is garbage collected when the system managing the interaction releases its reference.

## Internal State & Concurrency
-   **State:**
    -   The outer RaycastSelector is effectively **immutable** after deserialization. Its configuration fields (distance, offset, etc.) are set once and are not modified during its lifetime.
    -   The inner RuntimeSelector is highly **mutable**. It stores the results of the last raycast in its `bestMatch` and `blockPosition` fields. This state is updated every time its `tick` method is called.

-   **Thread Safety:**
    -   The outer RaycastSelector is **thread-safe** and can be shared across multiple systems due to its immutable nature.
    -   The inner RuntimeSelector is **not thread-safe**. It is designed to be exclusively owned and operated by the single-threaded server game loop for a specific entity. Its methods must be called in a strict sequence (`tick`, then `selectTarget...`) within the same thread to ensure state consistency.

    **WARNING:** Accessing a RuntimeSelector instance from multiple threads will lead to race conditions and unpredictable behavior. It must not be shared across different entity processing contexts.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| newSelector() | Selector | O(1) | Factory method. Creates and returns a new stateful RuntimeSelector instance. |
| selectTargetPosition(cb, ref) | Vector3d | O(1) | Calculates the world-space starting position for the raycast based on the entity's transform and the configured offset. |
| toPacket() | com.hypixel.hytale.protocol.Selector | O(1) | Serializes the configuration into a network packet, used to inform the client about this selector's properties for prediction or rendering. |
| **RuntimeSelector.tick(...)** | void | O(N) | **Core Logic.** Executes the raycast into the world, checking for block and entity intersections. N is the number of entities in the search volume. Updates internal state with the closest valid target. |
| **RuntimeSelector.selectTargetEntities(...)** | void | O(1) | Consumes the entity result from the last `tick` call. Invokes the consumer callback if a valid entity was found. |
| **RuntimeSelector.selectTargetBlocks(...)** | void | O(1) | Consumes the block result from the last `tick` call. Invokes the consumer callback if a valid block was found. |

## Integration Patterns

### Standard Usage
The intended pattern is to create a RuntimeSelector for an interaction, update it every tick, and then query the results. This ensures the targeting is responsive to changes in the game world.

```java
// 1. Get a configured RaycastSelector (usually from an asset)
RaycastSelector config = getSelectorFromAsset();

// 2. Create a stateful runtime instance for a specific interaction
Selector runtimeSelector = config.newSelector();

// 3. In the game loop for the entity...
public void onEntityTick(CommandBuffer<EntityStore> cb, Ref<EntityStore> entity) {
    // Update the selector's state with the latest world information
    runtimeSelector.tick(cb, entity, time, runTime);

    // Query the results and apply game logic
    runtimeSelector.selectTargetEntities(cb, entity, (targetRef, hitPosition) -> {
        // Logic for when an entity is targeted
        System.out.println("Targeted entity: " + targetRef.getIndex());
    }, filter);

    runtimeSelector.selectTargetBlocks(cb, entity, (x, y, z) -> {
        // Logic for when a block is targeted
        System.out.println("Targeted block at: " + x + "," + y + "," + z);
    });
}
```

### Anti-Patterns (Do NOT do this)
-   **State Staleness:** Calling `selectTargetEntities` or `selectTargetBlocks` multiple times without calling `tick` in between. The results will be stale and not reflect the current world state.
-   **Improper Sequencing:** Calling `selectTargetEntities` or `selectTargetBlocks` *before* the first `tick` call. The internal state will be uninitialized, yielding no results.
-   **Instance Re-use:** Using the same RuntimeSelector instance for multiple different entities simultaneously. The internal state will be overwritten, leading to incorrect targeting for all but one entity. Create a new instance for each independent interaction.

## Data Pipeline
The RaycastSelector processes data by querying the world state based on an entity's current transform and direction, storing the result internally for subsequent retrieval.

> Flow:
> Entity State (TransformComponent, HeadRotation) -> **RaycastSelector.tick()** -> World Query (Blocks & Entities) -> Collision & Distance Checks -> **Internal State (bestMatch, blockPosition)** -> selectTargetEntities() / selectTargetBlocks() -> Consumer Callback

