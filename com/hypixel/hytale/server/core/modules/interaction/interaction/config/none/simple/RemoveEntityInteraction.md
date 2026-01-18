---
description: Architectural reference for RemoveEntityInteraction
---

# RemoveEntityInteraction

**Package:** com.hypixel.hytale.server.core.modules.interaction.interaction.config.none.simple
**Type:** Transient

## Definition
```java
// Signature
public class RemoveEntityInteraction extends SimpleInstantInteraction {
```

## Architecture & Concepts
The RemoveEntityInteraction is a data-driven, server-side component responsible for executing the logic to despawn a single entity from the world. It is a concrete implementation of the abstract SimpleInstantInteraction, designed to be configured and deserialized from game data files rather than being hard-coded.

Architecturally, this class acts as a command object within the server's broader Interaction System. It encapsulates a single, atomic action: entity removal. Its primary role is to bridge a high-level game event (like using an item or triggering a script) with the low-level world modification APIs.

Key design principles include:
*   **Data-Driven:** The behavior, specifically the target of the removal, is defined by the `entityTarget` field. This field is populated by the `CODEC` during asset loading, allowing designers to create "Remove Entity" effects in configuration files without writing any Java code.
*   **Transactional Integrity:** The class does not directly manipulate the world state. Instead, it submits a removal command to the world's `CommandBuffer`. This ensures that the operation is queued and executed safely within the main server tick, preventing race conditions and maintaining a consistent world state.
*   **Safety:** A critical built-in safety check prevents the removal of any entity that has a Player component. This is a non-configurable guardrail to avoid accidentally despawning active players through this interaction.

## Lifecycle & Ownership
-   **Creation:** Instances are created exclusively by the `BuilderCodec` system during server startup or when game assets are loaded. A game designer would define this interaction in a configuration file (e.g., JSON), and the static `CODEC` field acts as the factory to deserialize that data into a live RemoveEntityInteraction object.
-   **Scope:** The object is transient and stateless with respect to the game world. An instance represents a configured *template* for an action. When an interaction is triggered, the engine uses this template to execute the `firstRun` method. The instance itself persists as long as the configuration that defines it is loaded.
-   **Destruction:** Managed by the Java garbage collector. It holds no native resources or persistent state that requires manual cleanup.

## Internal State & Concurrency
-   **State:** The class holds one piece of mutable state: the `entityTarget` field. However, this field is only intended to be set once during deserialization by the `CODEC`. After initialization, the object should be treated as immutable.
-   **Thread Safety:** The class is not inherently thread-safe for modification. However, its execution pattern is designed for a concurrent environment. The core logic within `firstRun` is safe because it defers the actual world modification to the main world thread by scheduling a task via `world.execute`. This is a critical pattern to ensure all entity modifications happen synchronously within the primary game loop, preventing corruption of the `EntityStore`.

## API Surface
The primary contract is its implementation of the `SimpleInstantInteraction` parent class, which is invoked by the server's interaction module.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| firstRun(type, context, cooldownHandler) | void | O(1) | Executes the entity removal logic. Determines the target entity, validates it is not a player, and schedules its removal on the main world thread. Throws NullPointerException if the context is invalid. |

## Integration Patterns

### Standard Usage
A developer or designer does not interact with this class via Java code. Instead, they define it declaratively within a game data file. The server's interaction system handles the instantiation and execution.

Example pseudo-configuration:
```json
{
  "id": "item.admin_wand",
  "interactions": [
    {
      "type": "hytale:remove_entity",
      "trigger": "PRIMARY_ACTION",
      "target": "LOOK_AT"
    }
  ]
}
```
In this example, an item is configured so that its primary action triggers a `RemoveEntityInteraction` targeting the entity the user is looking at.

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Do not use `new RemoveEntityInteraction()`. The object will be unconfigured, with a default `entityTarget` of USER, and will bypass the intended data-driven design. Always define interactions in configuration files.
-   **Manual Execution:** Do not call the `firstRun` method directly. It relies on a valid `InteractionContext` provided by the interaction module. Calling it manually with a null or incomplete context will result in `NullPointerException`s and unpredictable behavior.
-   **State Mutation:** Do not modify the `entityTarget` field after the object has been initialized by the `CODEC`. This can lead to inconsistent behavior across the server.

## Data Pipeline
The flow of data and control for this interaction follows a clear, asynchronous command pattern.

> Flow:
> Player Input or Scripted Event -> Interaction Module resolves the configured `RemoveEntityInteraction` -> `firstRun` is invoked with a valid `InteractionContext` -> Target entity is resolved via `entityTarget.getEntity()` -> A removal command (lambda) is scheduled via `world.execute()` -> On the next server tick, the main world thread executes the lambda -> `store.removeEntity()` is called, safely removing the entity from the `EntityStore`.

