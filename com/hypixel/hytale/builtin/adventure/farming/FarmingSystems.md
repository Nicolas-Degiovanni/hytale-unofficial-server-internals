---
description: Architectural reference for FarmingSystems
---

# FarmingSystems

**Package:** com.hypixel.hytale.builtin.adventure.farming
**Type:** Utility

## Definition
```java
// Signature
public class FarmingSystems {
    // Contains multiple nested static System classes
}
```

## Architecture & Concepts

FarmingSystems is not a singular object but a static container class that groups a suite of specialized systems within the Entity Component System (ECS) framework. Collectively, these systems orchestrate the entire lifecycle and simulation logic for all farming-related mechanics, including crop growth, soil hydration and decay, and animal coop population management.

The architecture follows a highly decoupled, event-driven model common in ECS.
- **Reactive Systems** (subclasses of RefSystem): These systems, such as OnFarmBlockAdded and OnCoopAdded, respond to the creation and destruction of specific block components. They are responsible for initialization, scheduling initial updates, and teardown logic. For example, when a player tills soil, a TilledSoilBlock component is created, which triggers OnSoilAdded to calculate and schedule its future decay time.
- **Proactive Systems** (subclasses of EntityTickingSystem): These systems, primarily the main Ticking class, drive the continuous simulation. The Ticking system iterates over active chunk sections, identifies blocks scheduled for an update, and dispatches the appropriate logic (tickSoil, tickCoop, tickFarming) based on the components attached to the block entity.

This design separates the *what* (the state, stored in Components like FarmingBlock) from the *how* (the logic, implemented in these Systems). The FarmingSystems class acts as a central namespace for this logic, ensuring that all rules governing the farming simulation are co-located and managed by the ECS engine.

### Lifecycle & Ownership
- **Creation:** The FarmingSystems class itself is never instantiated. Its nested static system classes are discovered and instantiated by the core ECS System Manager during server bootstrap.
- **Scope:** Each individual system persists for the entire server session. They are stateless services that operate on component data.
- **Destruction:** The systems are destroyed when the server shuts down and the ECS world is torn down.

## Internal State & Concurrency
- **State:** The FarmingSystems class and its nested systems are fundamentally **stateless**. All mutable data related to farming is stored in Components (e.g., FarmingBlock, TilledSoilBlock, CoopResidentComponent) attached to entities or in global Resources (e.g., WorldTimeResource).
- **Thread Safety:** The systems are designed to be run by the Hytale ECS scheduler. Concurrency is managed by the framework. Systems operate on isolated data sets defined by their queries. Modifications to the world state are not performed directly; instead, they are queued into a CommandBuffer. These commands are executed in a synchronized phase at the end of the system's execution, preventing race conditions.

**WARNING:** Note the use of `world.execute` within the Ticking system's tickCoop method. This schedules work to be run on the main world thread, which is a necessary pattern for complex queries like sphere-based entity lookups that might cross chunk boundaries and require a broader, consistent view of the world state.

## API Surface

The public contract of FarmingSystems is the set of systems it provides to the ECS engine. Direct invocation is not intended.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| CoopResidentEntitySystem | RefSystem | O(1) | Reacts to CoopResidentComponent removal to notify the parent CoopBlock. |
| CoopResidentTicking | EntityTickingSystem | O(N) | Periodically checks and despawns coop residents marked for removal. |
| OnCoopAdded | RefSystem | O(1) | Initializes a newly placed CoopBlock by scheduling its first tick. |
| OnFarmBlockAdded | RefSystem | O(1) | Initializes a new FarmingBlock with default state and schedules its first growth tick. |
| OnSoilAdded | RefSystem | O(1) | Initializes a new TilledSoilBlock by scheduling its decay tick. |
| Ticking | EntityTickingSystem | O(N) | The primary simulation loop. Iterates all ticking blocks in a chunk and dispatches to specialized tick logic. |

## Integration Patterns

### Standard Usage

Developers do not interact with FarmingSystems directly. The engine automatically discovers and integrates these systems. The pattern is implicit: creating an entity with a specific component (e.g., FarmingBlock) will cause the corresponding system (e.g., OnFarmBlockAdded) to be invoked by the engine for that entity.

The logic within these systems integrates with other core engine modules:
- **BlockModule:** To get positional information and component references for blocks.
- **WorldTimeResource:** To make time-dependent decisions for growth, decay, and spawning.
- **CommandBuffer:** To safely queue all world modifications, such as changing a block type or removing an entity.

```java
// A hypothetical system that creates a new farm plot.
// The engine handles the rest.
public void createFarmPlot(CommandBuffer<ChunkStore> commands, WorldChunk chunk, Vector3i pos) {
    // Setting the block type to one with farming data will cause the
    // BlockModule to add a FarmingBlock component.
    int farmBlockId = BlockType.getAssetMap().getIndex("my_game:wheat_stage_0");
    BlockType farmBlockType = BlockType.getAssetMap().getAsset(farmBlockId);
    chunk.setBlock(pos.x, pos.y, pos.z, farmBlockId, farmBlockType, 0, 0, 0);

    // At the end of the tick, the CommandBuffer will be flushed.
    // The new block entity with its FarmingBlock component will be created.
    // The FarmingSystems.OnFarmBlockAdded system will then automatically run for this new entity.
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never create an instance of a system using `new FarmingSystems.Ticking()`. The ECS engine is responsible for the lifecycle of all systems.
- **Stateful Systems:** Do not add member variables to these system classes to store state. All state must reside in Components or Resources to ensure proper serialization and concurrency.
- **Direct World Modification:** Avoid modifying world state (e.g., calling `worldChunk.setBlock`) directly from within a system's tick method. Always queue these operations on the provided CommandBuffer to ensure thread safety and deterministic execution.

## Data Pipeline

The systems form a processing pipeline driven by game events and the passage of time.

**Example 1: A Crop Growing**
> Flow:
> Engine Tick -> **FarmingSystems.Ticking** -> Iterates ticking blocks -> Finds a FarmingBlock ready to grow -> Calls FarmingUtil.tickFarming -> FarmingBlock component state is updated -> CommandBuffer queues a `setBlock` command to change wheat from stage 1 to stage 2.

**Example 2: A Coop Spawning an Animal**
> Flow:
> Engine Tick -> **FarmingSystems.Ticking** -> Finds a ticking CoopBlock -> Calls tickCoop -> Checks WorldTimeResource -> Determines it is daytime and residents should be outside -> `coopBlock.ensureSpawnResidentsInWorld` is called -> CommandBuffer queues an `addEntity` command for a new chicken entity with a CoopResidentComponent.

**Example 3: Tilled Soil Decaying**
> Flow:
> Player tills dirt -> `setBlock` call creates a TilledSoilBlock component -> **FarmingSystems.OnSoilAdded** runs -> Calculates decay time from block assets -> Schedules a future block tick -> Time passes -> **FarmingSystems.Ticking** runs -> Finds the ticking TilledSoilBlock -> Calls tickSoil -> Determines decay time has passed -> CommandBuffer queues a `setBlock` command to revert the block to regular dirt.

