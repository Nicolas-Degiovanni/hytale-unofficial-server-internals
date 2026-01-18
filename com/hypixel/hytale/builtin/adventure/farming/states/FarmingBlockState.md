---
description: Architectural reference for FarmingBlockState
---

# FarmingBlockState

**Package:** com.hypixel.hytale.builtin.adventure.farming.states
**Type:** Component (Data)

## Definition

**WARNING:** This component is deprecated and marked for removal. It represents a legacy implementation of the farming system. Do not use it for new development and plan for its replacement in existing systems.

```java
// Signature
@Deprecated(forRemoval = true)
public class FarmingBlockState implements Component<ChunkStore> {
```

## Architecture & Concepts

FarmingBlockState is a server-side data component that encapsulates the state of a single, growable block within the world, such as a crop. It functions as a Plain Old Java Object (POJO) whose lifecycle and data are managed by higher-level engine systems.

As an implementation of the Component interface, its primary role is to attach state to a game entity, in this case, a block within a ChunkStore. It holds all necessary data for the farming simulation to track growth, including the crop type, current growth stage, and timing information.

The core of its persistence mechanism is the static CODEC field. This `BuilderCodec` defines the contract for serializing the component's state to disk and deserializing it back into a live object when a world chunk is loaded. All game logic for farming is external to this class; systems like a hypothetical FarmingSystem would query and mutate instances of FarmingBlockState to simulate crop growth over time.

The public visibility of its fields is intentional, indicating it is a pure data container designed for high-performance access by a trusted, single-threaded game loop.

## Lifecycle & Ownership

-   **Creation:** Instances are not created directly via the constructor. They are instantiated by the server's persistence layer when a world chunk is loaded from disk. The `CODEC` reads the saved data and populates a new FarmingBlockState object. New instances are also created programmatically by game systems when a player plants a new crop.
-   **Scope:** The in-memory lifetime of a FarmingBlockState instance is strictly tied to its parent world chunk. It exists only while the chunk is loaded and active on the server.
-   **Destruction:** The object is dereferenced and becomes eligible for garbage collection when the corresponding world chunk is unloaded. Prior to unloading, its state is serialized to disk via the `CODEC`.

## Internal State & Concurrency

-   **State:** The state is entirely mutable. All fields are public and are directly modified by the server's farming simulation logic during the game tick. It serves as the in-memory, "live" version of a farming block's persistent data.
-   **Thread Safety:** **This class is not thread-safe.** It is designed to be accessed and modified exclusively by the main server thread responsible for the game world's simulation. Any concurrent modification from other threads (e.g., network threads, background workers) without external synchronization on the parent chunk will result in data corruption, race conditions, and undefined behavior.

## API Surface

The primary API is direct field access. The following symbols represent the most critical aspects of its public contract. Standard getters and setters are omitted for brevity.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| CODEC | BuilderCodec | - | **Critical:** Static serializer/deserializer for persistence. |
| baseCrop | String | O(1) | The unique identifier for the type of crop. |
| stageStart | Instant | O(1) | The timestamp when the current growth stage began. |
| currentFarmingStageIndex | int | O(1) | The zero-based index of the current growth stage. |
| clone() | Component | O(1) | **Warning:** This method is dangerously named. It does not create a copy and instead returns the original instance (`this`). |

## Integration Patterns

### Standard Usage

A server-side system responsible for game logic accesses this component through a block's data container. The system reads the state, applies simulation rules, and writes back the updated state.

```java
// A hypothetical FarmingSystem processing a block
void processFarmingTick(Block block) {
    FarmingBlockState state = block.getComponent(FarmingBlockState.class);
    if (state == null) {
        return; // Not a farming block
    }

    // Logic to check if enough time has passed to advance the growth stage
    long timeSinceStageStart = Duration.between(state.stageStart, Instant.now()).getSeconds();
    if (timeSinceStageStart > getRequiredGrowthTime(state.baseCrop)) {
        state.currentFarmingStageIndex++;
        state.stageStart = Instant.now();
        block.markDirty(); // Notify the engine of a state change for persistence
    }
}
```

### Anti-Patterns (Do NOT do this)

-   **Direct Instantiation:** Never call `new FarmingBlockState()`. The component must be created and managed by the world engine to ensure it is correctly registered and persisted.
-   **Misuse of clone:** Do not call `clone()` expecting a new, independent copy of the state. It returns the same object, which will lead to unintended shared state modifications.
-   **Asynchronous Modification:** Do not read or write to the fields of this component from any thread other than the one that owns the corresponding world region or chunk. This will bypass engine safeguards and corrupt world data.

## Data Pipeline

The FarmingBlockState acts as a conduit between persistent storage and the live game simulation. Its data flows through the engine in a well-defined cycle.

> Flow (Load -> Simulate -> Save):
> World Save File on Disk -> ChunkStore Deserializer (using CODEC) -> **FarmingBlockState** (In-Memory) -> Farming System (Reads/Writes State) -> ChunkStore Serializer (using CODEC) -> World Save File on Disk

