---
description: Architectural reference for NoiseOperation
---

# NoiseOperation

**Package:** com.hypixel.hytale.builtin.buildertools.tooloperations
**Type:** Transient

## Definition
```java
// Signature
public class NoiseOperation extends ToolOperation {
```

## Architecture & Concepts
The NoiseOperation class is a server-side, stateful command object that encapsulates the logic for a specific builder tool effect: applying a randomized, sparse pattern of blocks. It functions as a *Strategy* in the context of the broader Builder Tools system. Each instance represents a single, atomic world modification event triggered by a player.

This operation is not a long-lived service. It is created on-demand to process a network packet, applies its logic to a volume of blocks defined by its parent ToolOperation, and is then discarded. Its primary purpose is to enable "texturing" or "weathering" effects by placing blocks only if the target location is empty, adjacent to existing geometry, and passes a random probability check based on the tool's configured density.

## Lifecycle & Ownership
- **Creation:** Instantiated by the server-side Builder Tools framework in direct response to receiving a BuilderToolOnUseInteraction packet from a client. The framework is responsible for supplying the necessary context, including the world reference, the player entity, and the interaction packet.
- **Scope:** The object's lifetime is exceptionally short, scoped to the execution of a single tool usage. It exists only for the duration required to iterate over the brush's volume and apply its changes.
- **Destruction:** The object becomes eligible for garbage collection immediately after the parent ToolOperation completes its execution. It holds no external references and is not registered with any service locator or registry.

## Internal State & Concurrency
- **State:** The NoiseOperation is stateful but effectively immutable after construction. The core parameter, brushDensity, is a final field initialized once. All other operational state, such as the world editor context and the block pattern, is inherited from its parent and is considered constant for the duration of the operation.
- **Thread Safety:** This class is **not thread-safe** and must be confined to the server's main game thread. It is designed to be created, used, and discarded within a single, synchronous tick cycle. All world modifications performed through the *edit* context are assumed to be handled by the underlying world storage system, which is responsible for its own concurrency control.

## API Surface
The primary contract is the protected `execute0` method, which is invoked by the parent `ToolOperation`'s execution loop.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| execute0(int x, int y, int z) | boolean | O(1) | Executes the noise logic for a single block coordinate. Returns true to indicate the operation should continue. |

## Integration Patterns

### Standard Usage
This class is not intended for direct use by developers. It is instantiated and managed exclusively by the internal Builder Tools system. The system iterates over a geometric shape (e.g., a sphere) and invokes `execute0` for each block within that volume.

```java
// Conceptual example of internal framework usage
ToolOperation operation = toolSystem.createOperationFromPacket(player, packet);

// The framework iterates over the brush volume
for (BlockPos pos : brush.getBlocks()) {
   // The parent's execute() method calls execute0() internally
   operation.execute(pos.getX(), pos.getY(), pos.getZ());
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never create an instance using `new NoiseOperation()`. The constructor requires a complex and precise state (world references, packets, accessors) that can only be provided correctly by the parent framework.
- **State Reuse:** Do not cache or reuse a NoiseOperation instance across multiple tool uses or game ticks. Each instance is tied to the specific context of a single network packet and must be recreated for every new operation.

## Data Pipeline
The NoiseOperation is a key processing step in the server-side handling of a player's build action. It translates a high-level user intent into low-level block changes.

> Flow:
> Player Client Input -> BuilderToolOnUseInteraction Packet -> Server Network Handler -> Builder Tool System -> **NoiseOperation Instantiation** -> WorldEdit Context -> Chunk Modification

