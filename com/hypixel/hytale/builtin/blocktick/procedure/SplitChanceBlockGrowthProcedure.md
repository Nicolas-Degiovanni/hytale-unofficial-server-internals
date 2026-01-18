---
description: Architectural reference for SplitChanceBlockGrowthProcedure
---

# SplitChanceBlockGrowthProcedure

**Package:** com.hypixel.hytale.builtin.blocktick.procedure
**Type:** Transient Configuration Object

## Definition
```java
// Signature
public class SplitChanceBlockGrowthProcedure extends BasicChanceBlockGrowthProcedure {
```

## Architecture & Concepts
The SplitChanceBlockGrowthProcedure is a server-side component that defines a specific rule for block state transitions during a world tick. It represents a probabilistic growth event where a block can transform into one of several potential outcomes, each with a distinct, weighted chance.

This class is a concrete implementation of the `TickProcedure` pattern. It is designed to be entirely data-driven, deserialized from block asset files. This decouples the game logic for block behavior from the core engine, allowing designers and modders to define complex, emergent growth systems without writing new Java code.

Architecturally, it acts as a stateless, executable rule within the server's block ticking system. When the world simulation decides to process a scheduled tick for a given block, it retrieves the associated procedure instance and executes it. This class extends BasicChanceBlockGrowthProcedure, inheriting a preliminary chance check before its own specialized logic is invoked. Its unique responsibility is to resolve the *outcome* of a successful growth event from a weighted list of possibilities.

## Lifecycle & Ownership
- **Creation:** Instances are not typically created programmatically via the constructor. The primary creation path is through the static CODEC field, which is used by the server's asset loading system to deserialize block definition files (in BSON format) into in-memory procedure objects at server startup.

- **Scope:** An instance of SplitChanceBlockGrowthProcedure lives as long as the server is running. It is loaded once, associated with a specific block definition, and stored in a central asset registry. It is effectively a singleton from the perspective of the block type it defines.

- **Destruction:** The object is garbage collected when the server shuts down and the asset registries are cleared.

## Internal State & Concurrency
- **State:** The internal state, consisting of the outcome block IDs (data) and their associated weights (chances), is populated once during deserialization. After creation, the object's state is **immutable**. The sumChances field is a pre-calculated cache to optimize the random selection process.

- **Thread Safety:** This class is designed to be thread-safe. Its immutable state allows it to be safely read by multiple world-simulation threads concurrently. The core execution relies on RandomUtil, which is expected to provide a thread-safe source of randomness, making the procedure safe for parallel execution across different world chunks.

## API Surface
The primary interaction with this class is through its execution by the block ticking system, not direct API calls. The constructor is public for programmatic assembly but is not the standard integration path.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| SplitChanceBlockGrowthProcedure(int, int, int[], String[], boolean) | constructor | O(N) | Programmatically creates a new procedure. Validates input arrays. Throws IllegalArgumentException on invalid input. |
| executeToBlock(World, int, int, int, String) | protected boolean | O(1) | Overrides parent logic. Performs a weighted random roll to select a block ID from its internal state, then delegates to the parent implementation to perform the world modification. |

## Integration Patterns

### Standard Usage
While the primary integration is data-driven via asset files, the procedure can be constructed programmatically for dynamic, in-code rule generation.

```java
// Example: Create a procedure where a block has a 70% chance to become
// "hytale:stone" and a 30% chance to become "hytale:cobblestone".
String[] outcomes = new String[]{"hytale:stone", "hytale:cobblestone"};
int[] weights = new int[]{70, 30};
boolean willTickAgain = false;

// The first two arguments are inherited from BasicChanceBlockGrowthProcedure,
// controlling the initial chance of the procedure running at all.
SplitChanceBlockGrowthProcedure procedure = new SplitChanceBlockGrowthProcedure(1, 1, weights, outcomes, willTickAgain);

// In a real scenario, this procedure would be executed by the engine, not directly.
// engine.executeBlockProcedure(world, x, y, z, procedure);
```

### Anti-Patterns (Do NOT do this)
- **Mismatched Arrays:** Do not call the constructor with `chances` and `data` arrays of different lengths. The constructor will throw an `IllegalArgumentException`, but this check should be performed by the calling code.
- **Post-Construction Mutation:** Never attempt to modify the internal `chances` or `data` arrays after the object has been created. The internal `sumChances` cache would become invalid, leading to unpredictable and incorrect behavior during execution.
- **Negative Chances:** Do not provide negative values in the `chances` array. This will result in an `IllegalArgumentException`.

## Data Pipeline
The flow of data from configuration to world-state change is central to this component's design.

> Flow:
> Block Asset File (BSON) -> Server Asset Loader -> **SplitChanceBlockGrowthProcedure.CODEC** -> In-Memory Procedure Instance -> Block Tick Scheduler -> `executeToBlock` -> World Database Update

