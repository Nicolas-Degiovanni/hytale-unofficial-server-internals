---
description: Architectural reference for SmootherOperation
---

# SmootherOperation

**Package:** com.hypixel.hytale.builtin.buildertools.tooloperations
**Type:** Transient

## Definition
```java
// Signature
public class SmootherOperation extends ToolOperation {
```

## Architecture & Concepts
The SmootherOperation is a concrete implementation of the **Strategy Pattern**, inheriting from the abstract ToolOperation. It encapsulates the logic for the "Smoother" builder tool, which is used to homogenize terrain by blending distinct block types.

This class acts as a server-side command object. It is created in direct response to a client-side user action, specifically a `BuilderToolOnUseInteraction` packet. Its sole responsibility is to translate the intent of that packet—to smooth a region of the world—into a series of specific block modifications.

The core concept revolves around a strength-based sampling algorithm. For any given block coordinate, the operation queries a helper system (`BuilderState`) to sample the surrounding blocks. This sample yields a dominant solid block, a dominant filler block, and a `solidStrength` value. The SmootherOperation then compares this sampled strength against its own calculated `strength` (derived from the tool's settings) to decide whether to place the solid or filler block, effectively eroding or accreting terrain to match its neighbors.

## Lifecycle & Ownership
- **Creation:** Instantiated by a factory method within the `BuilderToolsPlugin` when a `BuilderToolOnUseInteraction` packet corresponding to the smoother tool is received and processed by the server. It is never created directly by general game systems.
- **Scope:** Extremely short-lived. An instance of SmootherOperation exists only for the duration of a single tool application. The parent `ToolOperation` class iterates over a volume of blocks, calling `execute0` for each one. Once this loop completes, the SmootherOperation instance is no longer referenced and becomes eligible for garbage collection.
- **Destruction:** The object is managed by the Java garbage collector. There are no native resources or explicit cleanup methods required.

## Internal State & Concurrency
- **State:** The primary internal state is the `strength` field, a float calculated once in the constructor. This value is determined by the tool's configuration (`AddStrength` or `RemoveStrength`) and the type of interaction (Primary or Secondary click). Once set, this state is effectively immutable for the object's lifetime. State related to the world modification itself is managed by the parent `ToolOperation` via the `edit` field.
- **Thread Safety:** **This class is not thread-safe and must not be treated as such.** It is designed to be created, used, and discarded exclusively on the main server thread that is responsible for world ticks and modifications. The underlying `WorldEdit` context it uses for block changes is not safe for concurrent access.

## API Surface
The public contract is defined by the parent `ToolOperation`. The key implementation logic is in the protected `execute0` method.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| execute0(int x, int y, int z) | boolean | O(k) | Applies the smoothing logic to a single block at the given coordinates. Complexity is dependent on the neighbor sampling algorithm in `BuilderState`. Returns true on successful modification. |

## Integration Patterns

### Standard Usage
This class is not intended for direct use by developers. It is invoked internally by the builder tools system. The system identifies the tool being used from the network packet and dispatches to the appropriate `ToolOperation` implementation.

```java
// Conceptual example from within the BuilderToolsPlugin

// 1. A network packet arrives
BuilderToolOnUseInteraction packet = ...;
Player player = ...;

// 2. The plugin determines the operation type and creates an instance
ToolOperation operation = new SmootherOperation(ref, player, packet, accessor);

// 3. The operation is executed over the affected area
operation.execute(); // This internally calls execute0 for each block
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never use `new SmootherOperation()` in game logic. The builder tool framework is responsible for its creation and lifecycle. Manually creating it will lead to unmanaged and unpredictable world edits.
- **State Reuse:** Do not cache and reuse an instance of SmootherOperation. It is stateful based on the initial packet and is designed to be single-use.
- **Off-Thread Execution:** Calling `execute0` or the parent `execute` from any thread other than the main server world thread will cause severe concurrency issues, race conditions, and potential world corruption.

## Data Pipeline
The flow of data from user input to world change is linear and synchronous with the server tick.

> Flow:
> Client Mouse Click -> `BuilderToolOnUseInteraction` Packet -> Server Network Layer -> `BuilderToolsPlugin` -> **`SmootherOperation` Instantiation** -> `ToolOperation.execute()` Loop -> `SmootherOperation.execute0()` -> `WorldEdit.setBlock()` -> World State Change & Network Sync

