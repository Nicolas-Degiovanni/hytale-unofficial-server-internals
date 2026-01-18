---
description: Architectural reference for MountInteraction
---

# MountInteraction

**Package:** com.hypixel.hytale.builtin.mounts.interactions
**Type:** Transient

## Definition
```java
// Signature
public class MountInteraction extends SimpleInstantInteraction {
```

## Architecture & Concepts
The MountInteraction class is a data-driven, server-side handler that implements the logic for one entity mounting another. It is a concrete implementation of the abstract `SimpleInstantInteraction`, fitting into the engine's broader Interaction System.

Architecturally, this class is not a service or a manager but a *behavioral configuration*. Its primary role is to be deserialized from asset files (e.g., JSON) via its static CODEC field. This allows game designers to define mountable behaviors for entities, specifying properties like attachment offsets and controller types, without writing or modifying Java code.

During gameplay, when a player or NPC initiates an interaction with an entity configured with a MountInteraction, the server's interaction module invokes the `firstRun` method. This method then orchestrates the state change by adding and removing components from the involved entities through a transactional `CommandBuffer`, ensuring atomicity within a single game tick.

### Lifecycle & Ownership
-   **Creation:** Instances are not created directly using the `new` keyword in game logic. They are instantiated by the engine's asset loading and codec system during server startup or when game definitions are loaded. The static `CODEC` field defines how to deserialize the configuration into a valid MountInteraction object.
-   **Scope:** An instance of MountInteraction is a stateless configuration object. It persists in memory for as long as the entity definition it is associated with is loaded. It can be considered a flyweight, as a single instance can be reused for all interactions of its type.
-   **Destruction:** The object is managed by the Java garbage collector. It is eligible for cleanup when the associated game assets are unloaded, typically during a server shutdown or a hot-reload of assets.

## Internal State & Concurrency
-   **State:** The internal state consists of `attachmentOffset` and `controller`. This state is populated once during deserialization and is treated as immutable thereafter. The `firstRun` method does not mutate the MountInteraction instance itself; it only reads this configuration to modify the state of game entities.
-   **Thread Safety:** The class is inherently thread-safe for reads. Since its internal state is effectively immutable after creation, a single instance can be safely referenced by multiple threads. The execution of the `firstRun` method is managed by the server's interaction module, which is responsible for ensuring that modifications to the Entity-Component-System via the `CommandBuffer` are synchronized and do not cause race conditions. The class itself contains no explicit locking mechanisms.

## API Surface
The primary contract is the `firstRun` method, inherited from its parent and invoked by the engine.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| firstRun(type, context, cooldownHandler) | void | O(1) | Executes the core mounting logic. This is the entry point called by the interaction system. It validates the state of both entities and queues component changes to mount or dismount. |

## Integration Patterns

### Standard Usage
This class is not intended to be used directly in procedural code. Instead, it is defined declaratively in an entity's configuration files. The engine's interaction system resolves and executes this logic automatically.

A conceptual entity definition might look like this:
```json
// Example in a hypothetical entity JSON file
{
  "id": "hytale:horse",
  "components": {
    // ... other components
  },
  "interactions": [
    {
      "type": "com.hypixel.hytale.builtin.mounts.interactions.MountInteraction",
      "AttachmentOffset": { "x": 0.0, "y": 1.5, "z": 0.2 },
      "Controller": "Player"
    }
  ]
}
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Do not use `new MountInteraction()`. This bypasses the data-driven configuration system and creates an unconfigured object that will likely fail.
-   **Manual Invocation:** Avoid calling the `firstRun` method directly. It requires a fully-formed `InteractionContext` which is only provided by the server's interaction module. Manual invocation will break state management and lead to desynchronization.
-   **State Mutation:** Do not attempt to modify the `attachmentOffset` or `controller` fields after the object has been created. These are intended to be immutable configuration values.

## Data Pipeline
The flow of data and control for a mounting action is managed by the server-side game loop and interaction systems.

> Flow:
> Player Input (Interact Key) -> Client Network Packet -> Server Interaction Request -> Interaction Module resolves target and finds **MountInteraction** -> `firstRun` is invoked with `InteractionContext` -> Component changes are queued in `CommandBuffer` -> CommandBuffer is flushed at end of tick -> ECS state is updated -> State changes are replicated to clients

