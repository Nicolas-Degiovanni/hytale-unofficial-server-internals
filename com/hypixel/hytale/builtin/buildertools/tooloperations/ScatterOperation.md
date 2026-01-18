---
description: Architectural reference for ScatterOperation
---

# ScatterOperation

**Package:** com.hypixel.hytale.builtin.buildertools.tooloperations
**Type:** Transient

## Definition
```java
// Signature
public class ScatterOperation extends ToolOperation {
```

## Architecture & Concepts
The ScatterOperation is a concrete implementation of the **Strategy Pattern**, inheriting from the abstract ToolOperation class. It encapsulates the specific logic for a "scatter" or "spray paint" builder tool functionality on the server.

This class is a short-lived command object responsible for modifying a single voxel based on a set of placement rules and a probabilistic check. Its primary role is to apply a block pattern to a surface in a non-uniform, organic-feeling manner. It directly interacts with a world editing context to place blocks, conditional on the properties of adjacent blocks and the tool's configured density.

It acts as the final step in the server-side execution chain for a player's build action, translating a network request into a direct world state modification.

### Lifecycle & Ownership
- **Creation:** Instantiated by a higher-level manager within the builder tools system, likely a factory, in direct response to a server receiving a BuilderToolOnUseInteraction packet from a client.
- **Scope:** Extremely short-lived. An instance of ScatterOperation exists only for the duration of a single tool application against a set of coordinates. It is created, executed, and then immediately becomes eligible for garbage collection.
- **Destruction:** The object is not explicitly destroyed. It is cleaned up by the JVM garbage collector once the parent execution scope (e.g., the tool-use processing loop) completes and releases its reference.

## Internal State & Concurrency
- **State:** The ScatterOperation is stateful for the duration of its execution. It holds references to the world editing session, the player's current tool configuration (via BuilderState), and the block pattern to apply. It also reads from a globally shared, static BlockTypeAssetMap, which is treated as immutable for the purposes of this operation.

- **Thread Safety:** **This class is not thread-safe and must not be shared across threads.** It is designed to be instantiated, used, and discarded within the context of a single, synchronized game-tick or world-processing thread. Its core function, `edit.setBlock`, is a mutating operation on the world state that is not internally protected against race conditions.

## API Surface
The primary contract is fulfilled by overriding the `execute0` method from its parent, ToolOperation.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| execute0(int x, int y, int z) | boolean | O(1) | Applies the scatter logic to a single voxel coordinate. Returns true on completion. The operation may or may not place a block based on internal rules and random chance. |

## Integration Patterns

### Standard Usage
This class is not intended for direct use by most developers. It is invoked by the server's builder tool system. The conceptual usage is as follows.

```java
// Conceptual example within a hypothetical ToolExecutor
ToolOperation operation = toolOperationFactory.create(player, packet); // Creates a ScatterOperation
for (BlockPos pos : affectedArea) {
   operation.execute(pos.getX(), pos.getY(), pos.getZ());
}
```

### Anti-Patterns (Do NOT do this)
- **State Reuse:** Do not attempt to cache and reuse a ScatterOperation instance for multiple distinct tool applications. Each instance is initialized with context-specific state from the network packet and is not designed to be re-executed.
- **Concurrent Execution:** Never call `execute0` from multiple threads on the same instance or on different instances targeting the same world region without external locking. This will lead to world corruption and severe race conditions.
- **Direct Instantiation:** Avoid constructing this class with `new ScatterOperation()`. The builder tool system is responsible for its creation and providing the correct, context-sensitive dependencies like the world edit session and player state.

## Data Pipeline
The ScatterOperation is a consumer of network data and a producer of world state changes.

> Flow:
> Player Input -> BuilderToolOnUseInteraction Packet -> Server Network Handler -> Builder Tool System -> **ScatterOperation.execute0()** -> WorldEditSession.setBlock() -> World State Change

