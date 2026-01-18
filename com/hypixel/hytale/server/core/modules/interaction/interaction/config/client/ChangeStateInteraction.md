---
description: Architectural reference for ChangeStateInteraction
---

# ChangeStateInteraction

**Package:** com.hypixel.hytale.server.core.modules.interaction.interaction.config.client
**Type:** Transient

## Definition
```java
// Signature
public class ChangeStateInteraction extends SimpleBlockInteraction {
```

## Architecture & Concepts
The **ChangeStateInteraction** is a server-side, data-driven component responsible for implementing a specific type of block interaction: transitioning a target block from one state to another. It is a concrete implementation of the Strategy Pattern, where **SimpleBlockInteraction** defines the contract for interactions that target a single block.

Architecturally, this class is not a service or a manager but a configurable *behavior*. It is designed to be defined entirely within asset configuration files (e.g., JSON or HOCON) and deserialized at runtime by the engine's asset loading system. This allows game designers to create complex stateful blocks (like levers, doors, or machines) without writing any Java code.

Its primary role is to bridge the gap between a player's action and a direct world state mutation. It encapsulates the logic for:
1.  Identifying the current state of a block.
2.  Looking up the corresponding new state from a predefined map.
3.  Calculating the new block asset required to represent that state.
4.  Issuing a command to the world engine to perform the block replacement.

## Lifecycle & Ownership
-   **Creation:** Instances of **ChangeStateInteraction** are not created manually using the *new* keyword. They are instantiated exclusively by the **BuilderCodec** system during server startup or when game assets are loaded. The engine reads a configuration file, and the static **CODEC** field orchestrates the deserialization and construction of the object.
-   **Scope:** An instance of this class is a stateless template that persists for the entire server session, or until assets are reloaded. It is effectively a singleton from the perspective of its configuration, but multiple distinct configurations can exist as separate objects.
-   **Destruction:** The object is eligible for garbage collection when the server's asset registry is cleared, typically during a server shutdown or a full asset reload.

## Internal State & Concurrency
-   **State:** The internal state, primarily the **stateKeys** map, is populated once during deserialization and is considered immutable thereafter. This class is a pure configuration holder and does not maintain any runtime state related to a specific interaction event.

-   **Thread Safety:** This class is not inherently thread-safe and must not be accessed from arbitrary threads. All interaction logic is designed to be executed on the main server thread for the corresponding world. The **interactWithBlock** method operates on a **CommandBuffer**, which is a critical concurrency pattern. This ensures that world modifications are queued and executed at a safe point within the server's tick cycle, preventing race conditions and world corruption.

## API Surface
The public contract is primarily defined by its parent class, **SimpleBlockInteraction**.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| interactWithBlock(...) | void | Low | Executes the state change logic. This is the core operational method. Throws exceptions if the world or chunk is in an invalid state. Fails silently by setting the interaction context to **Failed** if a valid state transition is not found. |

## Integration Patterns

### Standard Usage
This class is not intended for direct use by developers. Its usage is declarative, defined within an asset file that describes an item or block's interactive properties. The engine's **InteractionModule** is responsible for invoking it.

A conceptual view of how the engine uses it:

```java
// ENGINE-LEVEL CODE (Conceptual)

// 1. An interaction is triggered for an entity.
InteractionDefinition def = entity.getActiveInteraction(); // Returns a ChangeStateInteraction instance

// 2. The engine validates and invokes the interaction against the target.
if (def instanceof ChangeStateInteraction) {
    // The engine provides the necessary world context.
    def.interactWithBlock(world, commandBuffer, type, context, ...);
}
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Never call **new ChangeStateInteraction()**. The object will be in an invalid state as its internal **stateKeys** map will be null. Always define interactions in data files to be loaded by the engine's **CODEC**.
-   **State Mutation:** Do not attempt to modify the **stateKeys** map after the object has been loaded. This violates the assumption of immutability and will lead to unpredictable behavior across the server.
-   **Asynchronous Invocation:** Do not call **interactWithBlock** from a separate thread. All world interactions must be synchronized with the main world tick, typically by submitting a task to the world's scheduler.

## Data Pipeline
The **ChangeStateInteraction** acts as a processing node in the server's input and world-update pipeline. Data flows through it to translate a high-level action into a low-level world mutation.

> Flow:
> Player Input Packet -> Server Network Layer -> InteractionModule -> **ChangeStateInteraction.interactWithBlock** -> CommandBuffer.setBlock -> WorldChunk Update -> (Optional) SoundUtil -> (Optional) Client-bound State Sync Packet

