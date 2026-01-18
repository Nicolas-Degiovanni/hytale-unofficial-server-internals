---
description: Architectural reference for CustomTemplateConnectedBlockPattern
---

# CustomTemplateConnectedBlockPattern

**Package:** com.hypixel.hytale.server.core.universe.world.connectedblocks
**Type:** Strategy Pattern Base

## Definition
```java
// Signature
public abstract class CustomTemplateConnectedBlockPattern {
```

## Architecture & Concepts
The CustomTemplateConnectedBlockPattern is an abstract base class that forms a cornerstone of the server's "Connected Blocks" system. This system is responsible for the dynamic visual transformation of blocks based on their immediate neighbors, enabling complex structures like fences, walls, and pipes to connect seamlessly.

This class defines the contract for a specific pattern matching strategy. Concrete implementations of this class encapsulate the logic for a single type of connection behavior, such as how a pillar block should change its model when stacked vertically or how a glass pane should form a corner.

It operates as a data-driven component. Specific patterns are not hard-coded but are defined in game asset files and deserialized at runtime using the provided CODEC. This allows designers to create new connected block behaviors without modifying engine code. The class is invoked by a CustomTemplateConnectedBlockRuleSet, which orchestrates the evaluation of a block's state against one or more patterns to determine its final appearance.

## Lifecycle & Ownership
- **Creation:** Instances of concrete subclasses are not instantiated directly via code. They are deserialized from game asset configuration files by the asset loading system during server initialization. The static CODEC field is the entry point for this data-driven instantiation, mapping a "Type" identifier in the asset file to a specific Java class implementation.
- **Scope:** These pattern objects are stateless and immutable configuration singletons. Once loaded from assets, they persist for the entire lifetime of the server. They are part of the static game definition data.
- **Destruction:** The objects are eligible for garbage collection only upon server shutdown when the asset registries are cleared.

## Internal State & Concurrency
- **State:** Immutable. A CustomTemplateConnectedBlockPattern instance contains no mutable state. Its purpose is to encapsulate logic, not to store runtime data. All necessary context for evaluation is passed as arguments to its methods.
- **Thread Safety:** The class itself is re-entrant and inherently thread-safe due to its stateless nature. However, its primary method, getConnectedBlockTypeKey, operates on a World object, which is a highly mutable and non-thread-safe data structure.

**WARNING:** Callers are responsible for ensuring that any calls to getConnectedBlockTypeKey are properly synchronized or executed within a thread context that has exclusive access to the relevant world chunk, such as the main server tick loop or a dedicated world generation thread. Failure to do so will result in severe concurrency violations and world corruption.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getConnectedBlockTypeKey(...) | Optional | O(1) | Evaluates the block at the given position against the pattern's logic. Reads a constant number of neighboring blocks. Returns a ConnectedBlockResult if the pattern matches, otherwise returns an empty Optional. |

## Integration Patterns

### Standard Usage
A developer or system integrator does not interact with this class directly. It is invoked automatically by the engine's block update logic. The primary integration is through asset definition, where a designer specifies which pattern to use for a given block type.

The engine's internal flow resembles the following:
```java
// PSEUDO-CODE: Engine-level invocation
// This logic resides within the Connected Blocks system, not in typical game code.

// 1. A block update occurs at 'position' in 'world'.
BlockType blockType = world.getBlock(position).getType();
CustomTemplateConnectedBlockRuleSet ruleSet = blockType.getConnectedBlockRuleSet();

if (ruleSet != null) {
    // 2. The rule set's pattern is invoked.
    CustomTemplateConnectedBlockPattern pattern = ruleSet.getPattern();
    Optional<Result> result = pattern.getConnectedBlockTypeKey(..., world, position, ...);

    // 3. If the pattern matched, the block state is updated.
    if (result.isPresent()) {
        world.setBlockState(position, result.get().getNewState());
    }
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never use `new ConcretePattern()`. The connected blocks system relies entirely on data-driven configuration. All patterns must be defined in asset files and loaded by the server's AssetManager.
- **Stateful Implementations:** Do not create concrete subclasses that store internal state. This violates the design contract and will lead to unpredictable behavior and severe concurrency issues, as a single pattern instance is shared across the entire server.
- **Unsafe World Access:** Do not call getConnectedBlockTypeKey from an asynchronous thread without acquiring the appropriate lock on the world or chunk. This will cause race conditions and corrupt world data.

## Data Pipeline
This class is a processing node in the block update pipeline. It reads data from the world state but does not, by itself, produce side effects.

> Flow:
> Block Update Event (Player action, world gen) -> Connected Blocks System -> CustomTemplateConnectedBlockRuleSet -> **CustomTemplateConnectedBlockPattern** -> World State Read (Neighboring blocks) -> ConnectedBlockResult -> World State Update System -> Network Packet to Client

