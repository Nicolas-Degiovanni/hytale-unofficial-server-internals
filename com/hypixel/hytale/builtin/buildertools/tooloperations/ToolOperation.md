---
description: Architectural reference for ToolOperation
---

# ToolOperation

**Package:** com.hypixel.hytale.builtin.buildertools.tooloperations
**Type:** Transient Command Object

## Definition
```java
// Signature
public abstract class ToolOperation implements TriIntObjPredicate<Void> {
```

## Architecture & Concepts

The ToolOperation class is the server-side abstraction for any discrete action performed by a player using a Builder Tool. It embodies the Command design pattern, encapsulating a single, complete world modification request into a stateful object. Its primary role is to translate a high-level user interaction, received as a network packet, into a specific set of block changes within the game world.

This class serves as the bridge between the player's intent and the world's state. When a player uses a tool, a `BuilderToolOnUseInteraction` packet is sent to the server. A static factory method, `fromPacket`, acts as the main entry point. It inspects the player's currently equipped tool and uses a static registry (`OPERATIONS`) to instantiate the correct concrete subclass, such as `PaintOperation` or `SculptOperation`.

The constructor is a critical component, gathering all necessary context into the object's final fields. This includes:
- The target world coordinates (x, y, z).
- The player entity and their current builder state.
- The active tool's configuration (brush shape, size, pattern, mask).
- Player-specific settings and transformations (rotation, mirroring).

A key architectural feature is its implementation of the `TriIntObjPredicate` functional interface. The `test` method contains the core logic to be applied to a single block coordinate. This allows generic shape iteration utilities (e.g., `BlockSphereUtil`, `BlockCubeUtil`) to traverse a geometric volume and invoke the `ToolOperation`'s logic for each block within that shape, effectively decoupling the shape's geometry from the operation's effect.

Each ToolOperation creates and owns an `EditOperation` instance, which is responsible for staging and applying the actual block changes to the world, providing a transactional boundary for the tool's effect.

### Lifecycle & Ownership
- **Creation:** A ToolOperation is instantiated exclusively through the static factory method `ToolOperation.fromPacket`. This occurs on the server when a `BuilderToolOnUseInteraction` packet is received from a client. The factory dispatches to the appropriate concrete constructor based on the active tool's ID.
- **Scope:** The object's lifetime is extremely short and bound to the processing of a single network packet. It is a transient, "fire-and-forget" object.
- **Destruction:** The object holds no external references after its `execute` method completes and is immediately eligible for garbage collection. It does not persist between tool uses.

## Internal State & Concurrency
- **State:** A ToolOperation is highly stateful but is effectively immutable after its construction. The constructor populates numerous final fields, creating a self-contained snapshot of the entire operation. It does not modify its own state after initialization.
- **Thread Safety:** This class is **not thread-safe** and is designed for single-threaded access within the context of a server-side entity processing loop. All interactions with an instance should occur on the same thread that created it.

  **WARNING:** While the instance itself is not thread-safe, it interacts with static, shared `ConcurrentHashMap` collections (`OPERATIONS`, `PROTOTYPE_TOOL_SETTINGS`). These collections are designed for concurrent access from multiple threads, ensuring that simultaneous tool usage by different players does not cause conflicts at the registry level. The primary concurrency concern, modifying the world, is delegated to the `World` and `EditOperation` systems, which are assumed to manage their own synchronization.

## API Surface
| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| fromPacket(ref, player, packet, accessor) | ToolOperation | O(1) | **Static Factory.** The primary entry point. Creates a concrete ToolOperation from a network packet. Throws Exception if no matching tool is found. |
| execute(accessor) | void | O(N) | Executes the operation at its configured coordinates. Complexity is proportional to the volume of the brush shape. |
| executeAt(x, y, z, accessor) | void | O(N) | Executes the operation at a specified coordinate, ignoring the one set during construction. |
| test(x, y, z, aVoid) | boolean | O(1) | The core logic callback. Applies the operation's effect to a single block. Not intended for direct invocation. |
| getEditOperation() | EditOperation | O(1) | Returns the associated object responsible for tracking world changes for this operation. |

## Integration Patterns

### Standard Usage
The ToolOperation is created and executed within the server's network packet handler. The lifecycle is self-contained and follows a clear pattern.

```java
// In a network packet handler for BuilderToolOnUseInteraction
try {
    // 1. Create the operation from the incoming packet and player state
    ToolOperation operation = ToolOperation.fromPacket(playerRef, player, packet, componentAccessor);

    // 2. Execute the operation to apply changes to the world
    operation.execute(componentAccessor);

    // 3. The operation object is now discarded
} catch (Exception e) {
    // Handle cases where the tool is not found or configured incorrectly
    log.error("Failed to execute tool operation", e);
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never use `new PaintOperation(...)` or any other concrete constructor directly. The constructor has a complex signature and relies on context provided by the `fromPacket` factory. Bypassing the factory will lead to an improperly configured object and runtime errors.
- **Instance Re-use:** Do not cache or re-use a ToolOperation instance. Each instance is a snapshot for a single user action. For a subsequent action, even if it is identical, a new ToolOperation must be created from a new packet.
- **State Mutation:** Do not attempt to modify the public or protected fields of a ToolOperation after it has been constructed. Its state is considered final and changing it will result in undefined behavior.

## Data Pipeline
The flow of data for a builder tool action is linear, starting from client input and resulting in a server-side world modification.

> Flow:
> Client Mouse Input -> `BuilderToolOnUseInteraction` Packet -> Server Network Handler -> `ToolOperation.fromPacket()` -> **ToolOperation Instance** -> `execute()` -> `executeShapeOperation()` -> `test()` callback per block -> `EditOperation` records changes -> World state updated

