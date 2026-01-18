---
description: Architectural reference for SmoothOperation
---

# SmoothOperation

**Package:** com.hypixel.hytale.builtin.buildertools.tooloperations
**Type:** Transient

## Definition
```java
// Signature
public class SmoothOperation extends ToolOperation {
```

## Architecture & Concepts
The SmoothOperation class is a concrete implementation of the Command Pattern, designed to execute a single, stateful "smooth" action on the game world. It is a server-side component within the Builder Tools subsystem, responsible for homogenizing terrain or structures by normalizing block types within a given volume.

This operation is instantiated in direct response to a client-initiated tool use event, encapsulated within a `BuilderToolOnUseInteraction` packet. Its core function is to analyze the block types in the immediate vicinity of a target coordinate. If the block at that coordinate is an outlier compared to the dominant block type in the sample area, it is replaced. This creates a "smoothing" or "blending" effect.

The class operates within a transactional context provided by its parent, ToolOperation, ensuring that all world modifications are part of a larger, atomic operation that can be managed by the server's world storage system.

### Lifecycle & Ownership
-   **Creation:** An instance of SmoothOperation is created by a factory method within the BuilderToolsPlugin. This occurs on the server when a `BuilderToolOnUseInteraction` packet is received and identified as a smooth action. The constructor immediately consumes packet data to configure its operational parameters, such as smoothing strength.
-   **Scope:** The object's lifetime is extremely short, scoped to a single tool application. It is created, its execution logic is invoked (potentially multiple times for each block in a targeted volume), and it is then discarded.
-   **Destruction:** The object becomes eligible for garbage collection as soon as the parent tool execution logic completes. It holds no references that would extend its life beyond the immediate task.

## Internal State & Concurrency
-   **State:** The class is effectively immutable after construction. The `smoothVolume` field is final and is calculated once from the initial network packet. All other operational state, such as the world edit accessor, is inherited from the ToolOperation base class and managed by the framework.
-   **Thread Safety:** This class is **not thread-safe** and must not be used concurrently. It is designed to be instantiated and executed exclusively on the server's main world-processing thread. Its methods perform direct, unsynchronized modifications to world state via the inherited `edit` field. Any off-thread access will lead to world corruption and server instability.

## API Surface
The primary contract is the protected `execute0` method, which is invoked by the public execution logic in the base class.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| execute0(int x, int y, int z) | boolean | O(1) | Executes the smoothing logic at a specific coordinate. Samples a fixed-size volume, determines the dominant block, and conditionally replaces the target block. This operation is considered constant time as the sample volume does not scale with input. |

## Integration Patterns

### Standard Usage
SmoothOperation is not intended for direct use by developers. It is instantiated and managed by the Builder Tools system. A higher-level executor iterates over a shape (e.g., a sphere) and invokes the operation for each block.

```java
// Conceptual example of how the framework uses this operation
// This code would exist within the Builder Tools system, not in user code.

// 1. Factory creates the operation from a packet
ToolOperation operation = OperationFactory.createFromPacket(packet);

// 2. Executor iterates over a volume and applies the operation
for (BlockCoordinate coord : brushShape.getCoordinates()) {
    operation.execute(coord.x, coord.y, coord.z);
}
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Never create an instance using `new SmoothOperation(...)`. The class requires a complex context (player reference, world edit accessor, packet data) that is only correctly assembled by the BuilderToolsPlugin framework.
-   **State Reuse:** Do not cache or reuse a SmoothOperation instance. Each instance is configured for a single, specific tool use event defined by the incoming network packet.
-   **Asynchronous Execution:** Never invoke `execute0` or any related methods from an asynchronous task or a different thread. All world modifications must be strictly serialized and performed on the main server thread to prevent race conditions and data corruption.

## Data Pipeline
The data flow for this operation begins with a client-side user action and results in a server-side world state change.

> Flow:
> Client Tool Use -> `BuilderToolOnUseInteraction` Packet -> Server Network Handler -> BuilderToolsPlugin -> **SmoothOperation Instantiation** -> World Edit Transaction -> World Storage Update -> State Replication to Clients

