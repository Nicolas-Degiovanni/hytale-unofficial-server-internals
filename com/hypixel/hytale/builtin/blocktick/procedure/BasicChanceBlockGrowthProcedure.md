---
description: Architectural reference for BasicChanceBlockGrowthProcedure
---

# BasicChanceBlockGrowthProcedure

**Package:** com.hypixel.hytale.builtin.blocktick.procedure
**Type:** Data-Driven Component

## Definition
```java
// Signature
public class BasicChanceBlockGrowthProcedure extends TickProcedure {
```

## Architecture & Concepts
The BasicChanceBlockGrowthProcedure is a concrete implementation of the TickProcedure strategy. It represents a single, stateless, probabilistic action that can be applied to a block during a server world tick. This class is a fundamental building block of the server's **Block Tick System**, which manages time-based changes to blocks, such as crop growth, vine spreading, or fire extinguishing.

Its primary architectural role is to decouple game logic from engine code. Instead of hard-coding block behaviors, designers can define them in external data files (e.g., JSON). The server's asset loading system uses the static CODEC field to deserialize these data definitions into executable BasicChanceBlockGrowthProcedure instances at runtime.

This component encapsulates a simple "roll the dice" logic: on each tick, it runs a chance-based check. If successful, it transforms the target block into a new block type, defined by the `to` field. This makes it ideal for non-deterministic growth or decay behaviors.

### Lifecycle & Ownership
-   **Creation:** Instances are not created directly via the `new` keyword in game logic. They are instantiated by the server's asset pipeline during the deserialization of block definition files. The `BuilderCodec` is responsible for constructing the object and populating its fields from the source data.
-   **Scope:** The lifecycle of a BasicChanceBlockGrowthProcedure instance is tied to the lifecycle of the block asset it is associated with. It is loaded into memory when the server starts or when assets are reloaded, and it persists as long as its parent block definition is active.
-   **Destruction:** The object is eligible for garbage collection when the server unloads the corresponding block assets, typically during a server shutdown or a full asset reload.

## Internal State & Concurrency
-   **State:** The internal state of this class (`chanceMin`, `chance`, `to`, `nextTicking`) is **effectively immutable**. Its fields are populated once at creation time by the CODEC and are not modified thereafter. This design ensures that block behavior is consistent and predictable based on its configuration.

-   **Thread Safety:** This class is **conditionally thread-safe**. Because its internal state is immutable, multiple threads can safely read its configuration. However, the `onTick` method is a mutating operation on an external `World` object. The Hytale engine's Block Tick Scheduler is responsible for ensuring that `onTick` is not called concurrently for the same `WorldChunk`, thus preventing race conditions at the world level. The `getRandom` method, inherited from the parent class, is assumed to be managed by the scheduler to ensure thread safety.

## API Surface
The public API is minimal, exposing only the core contract required by the Block Tick System.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| onTick(world, wc, x, y, z, id) | BlockTickStrategy | O(1) | Executes the procedure for a specific block. Returns a strategy hint to the scheduler, indicating whether the block should continue ticking or be put to sleep. |

## Integration Patterns

### Standard Usage
This class is not intended to be used directly from Java code. Instead, it is configured declaratively within a block's asset definition file. The engine's Block Tick Scheduler invokes it automatically.

**Conceptual Block Asset Configuration (e.g., JSON):**
```json
{
  "id": "mygame:sapling",
  "tickProcedure": {
    "type": "BasicChanceBlockGrowthProcedure",
    "ChanceMin": 1,
    "Chance": 10,
    "NextId": "mygame:small_tree",
    "NextTicking": false
  }
}
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Do not use `new BasicChanceBlockGrowthProcedure()`. The behavior of a block is defined by its data assets. Bypassing the asset system to create instances will lead to inconsistent and unmanageable game logic.
-   **State Mutation:** Do not attempt to modify the fields of a procedure instance after it has been loaded. This violates the immutable state contract and can cause unpredictable behavior for all blocks using that definition.

## Data Pipeline
The BasicChanceBlockGrowthProcedure acts as a processing node in the server's block update pipeline. It translates declarative configuration data into imperative world state changes.

> Flow:
> Block Asset File -> Server Asset Loader -> **BasicChanceBlockGrowthProcedure** (Deserialized via CODEC) -> Block Tick Scheduler -> `onTick` Invocation -> `World.setBlock` Call -> World State Change

