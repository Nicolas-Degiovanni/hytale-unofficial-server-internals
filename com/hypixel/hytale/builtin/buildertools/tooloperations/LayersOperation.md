---
description: Architectural reference for LayersOperation
---

# LayersOperation

**Package:** com.hypixel.hytale.builtin.buildertools.tooloperations
**Type:** Transient

## Definition
```java
// Signature
public class LayersOperation extends ToolOperation {
```

## Architecture & Concepts
The LayersOperation class is a concrete implementation of the **Strategy Pattern**, where the abstract **ToolOperation** defines the contract for a single, stateful world modification command initiated by a player. This class encapsulates the specific logic for the "Layers" builder tool, which is designed to paint multi-layered materials onto existing surfaces.

Architecturally, it serves as a server-side command object. It is instantiated in direct response to a client-side network packet (**BuilderToolOnUseInteraction**) and is responsible for translating the tool's configuration parameters—such as layer depth, material patterns, and direction—into a series of block modifications within a **WorldEditSession**. Its primary function is to find the first solid block along a given vector and replace it with a new block, but only if there is empty space behind it within a specified maximum depth. This creates the effect of "painting" a layer of a defined thickness onto a surface.

## Lifecycle & Ownership
- **Creation:** An instance of LayersOperation is created by a higher-level handler within the builder tools system when a **BuilderToolOnUseInteraction** packet is received from the client. The factory or handler responsible for this is passed all necessary context, including the player entity, the world reference (**EntityStore**), and the raw packet data.

- **Scope:** The object's lifetime is exceptionally short, scoped to the processing of a single tool usage event. It holds the state for one "click" or "paint stroke" of the builder tool.

- **Destruction:** The object is eligible for garbage collection immediately after the builder tool handler has finished iterating over the brush's area of effect and has called **execute0** for each relevant block position. It is not reused or pooled.

## Internal State & Concurrency
- **State:** The object's configuration state is effectively immutable. All parameters from the tool packet (e.g., **layerOneLength**, **depthDirection**, **layerOneBlockPattern**) are stored in final fields upon construction. The only mutable state is the private boolean flag **failed**, which is used to short-circuit execution if the initial configuration is invalid (e.g., enabling layer three without layer two).

- **Thread Safety:** This class is **not thread-safe** and is designed to be confined to the server's main tick thread. All interactions, from creation to execution, must occur within the same synchronous block of logic that processes the player's input packet. Accessing an instance from multiple threads will lead to race conditions, particularly with the underlying **WorldEditSession**.

## API Surface
The primary interaction with this class is through its constructor and the overridden **execute0** method from its parent.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| LayersOperation(ref, player, packet, accessor) | constructor | O(1) | Instantiates and configures the operation from a network packet. May set the internal **failed** flag if validation fails. |
| attemptSetBlock(x, y, z, depth) | boolean | O(1) | Places a block at the given coordinates based on the depth. Selects the material from the appropriate layer's **BlockPattern**. Returns false if no layer is defined for the given depth. |

## Integration Patterns

### Standard Usage
This class is not intended to be instantiated or used directly by developers. It is created and managed by the server's builder tool system. The system applies the operation over a brush shape.

```java
// Conceptual example of how the system uses this class
// This code would exist within a generic builder tool handler.

// 1. Factory creates the correct operation from the packet
ToolOperation operation = ToolOperationFactory.create(player, packet);

// 2. The system iterates over the brush shape
for (Vector3i pos : brush.getPositions()) {
    // 3. The operation's logic is executed for each block
    // The public execute() in the parent class calls the internal execute0()
    operation.execute(pos.x, pos.y, pos.z);
}

// 4. The operation is now out of scope and will be garbage collected.
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not call **new LayersOperation()**. The class requires a very specific context (player, world, packet) that is only available within the server's tool handling logic.

- **State Reuse:** Do not hold a reference to a LayersOperation instance after its initial use. Its configuration is tied to a single packet and is not designed to be reused or reconfigured.

- **Asynchronous Execution:** Never invoke methods on this class from a separate thread. All world edit operations must be synchronized with the main server tick.

## Data Pipeline
The flow of data for a Layers operation begins with player input and ends with a world state change.

> Flow:
> Player uses tool in client -> **BuilderToolOnUseInteraction** packet sent to server -> Server-side tool handler decodes packet -> **LayersOperation** is instantiated with packet data -> **execute0** is called for each block in the brush -> **WorldEditSession.setBlock** is called -> World data is modified and propagated to clients.

