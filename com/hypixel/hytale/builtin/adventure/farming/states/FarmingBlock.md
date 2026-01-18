---
description: Architectural reference for FarmingBlock
---

# FarmingBlock

**Package:** com.hypixel.hytale.builtin.adventure.farming.states
**Type:** Data Component (Transient)

## Definition
```java
// Signature
public class FarmingBlock implements Component<ChunkStore> {
```

## Architecture & Concepts
The FarmingBlock is a data component, not a service or manager. Within Hytale's component-based architecture, it serves as a state container that attaches farming-specific attributes to a block within a world chunk. Its sole responsibility is to hold the data necessary for the farming simulation, such as growth progress, the last time the simulation was ticked, and genetic information like its generation.

This class is a fundamental part of the **FarmingPlugin**, which acts as the "System" in an Entity-Component-System (ECS) pattern. The FarmingPlugin contains the logic that reads, updates, and processes instances of FarmingBlock during server game ticks to simulate plant growth, spreading, and harvesting.

A critical feature is the static **CODEC** field. This defines the serialization and deserialization contract for the component, enabling its state to be persisted to disk as part of the world save data. The codec is optimized to omit default values, reducing storage footprint.

## Lifecycle & Ownership
- **Creation:** A FarmingBlock instance is created by the FarmingPlugin when a player action (like planting a seed) or a world generation event transforms a regular block into a farmable one. The newly created component is then attached to the block's data container within the parent ChunkStore.

- **Scope:** The component's lifetime is strictly tied to the block it represents. It persists across server restarts via the serialization mechanism. It exists only as long as the block is considered a farmable entity.

- **Destruction:** The component is detached and marked for garbage collection when the associated block is destroyed, harvested, or otherwise ceases to be a farmable plant. This cleanup is managed exclusively by the FarmingPlugin.

## Internal State & Concurrency
- **State:** The FarmingBlock is inherently **mutable**. Its primary purpose is to have its state, such as growthProgress and lastTickGameTime, continuously modified by the farming system over its lifetime.

- **Thread Safety:** This component is **not thread-safe** and must be considered thread-hostile. All interactions, both read and write, must be performed exclusively on the main server thread that executes the game tick. Unsynchronized access from other threads will result in data corruption, race conditions, and unpredictable game behavior.

## API Surface
The public API consists primarily of simple data accessors.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getComponentType() | static ComponentType | O(1) | Retrieves the globally registered ComponentType for FarmingBlock. |
| clone() | Component | O(1) | Creates a new FarmingBlock instance with an identical copy of the current state. |
| getGrowthProgress() | float | O(1) | Returns the current growth progress, typically a value between 0.0 and 1.0. |
| setGrowthProgress(float) | void | O(1) | Updates the growth progress. Must only be called by the owning system. |
| getLastTickGameTime() | Instant | O(1) | Returns the timestamp of the last game tick that processed this block. |
| setLastTickGameTime(Instant) | void | O(1) | Sets the timestamp of the last processing tick. |

## Integration Patterns

### Standard Usage
Direct interaction with FarmingBlock is typically reserved for the core farming system. Other systems should interact with it via higher-level APIs or by querying block state. The correct pattern involves retrieving the component from a world object, modifying it, and assuming the changes will be processed and persisted by the engine.

```java
// Example from within a hypothetical game system
// Note: World and BlockPosition are illustrative
Block targetBlock = world.getBlockAt(new BlockPosition(10, 20, 30));
FarmingBlock farmState = targetBlock.getComponent(FarmingBlock.getComponentType());

if (farmState != null && farmState.getGrowthProgress() >= 1.0f) {
    // Trigger harvest logic
    world.harvest(targetBlock);
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never create an instance using `new FarmingBlock()`. The component's lifecycle is managed by the FarmingPlugin. Manually created instances will not be attached to any block, will not be ticked by the game loop, and will not be persisted.

- **State Caching:** Do not retrieve a FarmingBlock component and store a reference to it across multiple game ticks. The block could be destroyed or modified at any time, invalidating the component instance. Always re-fetch the component from the block or world on the tick you need to interact with it.

- **Asynchronous Modification:** Never modify a FarmingBlock from a separate thread. All mutations must be queued as a task to be executed on the main server thread to prevent data corruption.

## Data Pipeline
The FarmingBlock is primarily a data payload that moves between the game simulation and persistent storage.

> **Serialization Flow (World Save):**
> Game Tick Logic (FarmingPlugin) -> Modifies **FarmingBlock** state -> Chunk Unload/Server Shutdown -> ChunkStore triggers persistence -> **FarmingBlock.CODEC** serializes object to binary -> World Save File

> **Deserialization Flow (World Load):**
> World Save File -> Chunk Load -> **FarmingBlock.CODEC** deserializes binary data -> New **FarmingBlock** instance created and populated -> Component attached to Block in ChunkStore -> Ready for Game Tick Logic

