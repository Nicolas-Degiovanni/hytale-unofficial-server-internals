---
description: Architectural reference for ChangeStatInteraction
---

# ChangeStatInteraction

**Package:** com.hypixel.hytale.server.core.modules.interaction.interaction.config.server
**Type:** Transient

## Definition
```java
// Signature
public class ChangeStatInteraction extends ChangeStatBaseInteraction {
```

## Architecture & Concepts
The ChangeStatInteraction class is a server-side **Interaction Configuration** object. It is not a long-running service but a data-driven definition of a discrete game action. Its sole purpose is to encapsulate the logic for modifying one or more statistics—such as health, mana, or speed—on a target entity.

This class acts as a critical bridge between high-level game design and the low-level server engine. Game designers define interactions in external asset files (e.g., JSON), and the server's `BuilderCodec` system deserializes these definitions into instances of ChangeStatInteraction.

When an in-game event triggers this interaction, its `firstRun` method is executed. This method uses a `CommandBuffer` to safely access and mutate the `EntityStatMap` component of the target entity, applying the pre-configured stat changes. This pattern ensures that game logic is data-driven, extensible, and decoupled from the core engine code.

## Lifecycle & Ownership
- **Creation:** Instances are created exclusively by the `BuilderCodec` deserialization framework when the server loads game content assets. Direct instantiation via the `new` keyword is a design violation and will result in a non-functional object.
- **Scope:** The lifetime of a ChangeStatInteraction instance is extremely short and stateless. It is held as part of a larger interaction definition. When the interaction is triggered, this object's logic is executed once, and it is then immediately eligible for garbage collection.
- **Destruction:** The object is managed by the Java garbage collector. As there are no persistent references to it after an interaction executes, it is typically cleaned up very quickly.

## Internal State & Concurrency
- **State:** **Effectively Immutable**. The object's fields, such as `entityStats` and `valueType`, are populated once during deserialization from an asset file. They are treated as read-only data for the duration of the object's lifecycle. Modifying this state at runtime is an anti-pattern and will lead to unpredictable behavior.
- **Thread Safety:** **Not Thread-Safe**. This class is designed to be executed exclusively on the main server thread that manages the game loop and entity updates. All operations, particularly the `firstRun` method which manipulates an entity's components via a `CommandBuffer`, must be synchronized with the primary server tick. Asynchronous or multi-threaded access will corrupt entity state.

## API Surface
The public API is intended for consumption by the server's internal interaction module, not for general developer use.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| firstRun(type, context, cooldownHandler) | void | O(N) | Executes the stat change logic. N is the number of stats to modify. This is the primary entry point for the interaction system. |
| generatePacket() | Interaction | O(1) | Creates a new, unconfigured network packet to inform clients of the stat change. |
| configurePacket(packet) | void | O(N) | Populates a network packet with the details of this interaction. N is the number of stats to modify. |

## Integration Patterns

### Standard Usage
A developer or content creator does not interact with this class directly in Java code. Instead, they define its properties within a game asset file, which the server then loads. The system handles the lifecycle and execution.

The following example shows the conceptual flow of how the **system** uses a pre-loaded instance.

```java
// PSEUDOCODE: Executed by the server's InteractionModule

// An instance deserialized from a game asset file
ChangeStatInteraction loadedInteraction = getInteractionFromConfig("player_heal_spell");

// When a player casts the spell, the system invokes the interaction
InteractionContext context = createInteractionContextForPlayer(player);
loadedInteraction.firstRun(InteractionType.PRIMARY, context, player.getCooldownHandler());
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never call `new ChangeStatInteraction()`. The object will lack the necessary data loaded by the codec and will fail or cause null pointer exceptions.
- **State Mutation:** Do not modify the public fields of this class after it has been loaded. This can cause inconsistent behavior between the server logic and the client-side prediction.
- **Asynchronous Execution:** Calling `firstRun` from a thread pool or any context outside the main server game loop is a critical error that will lead to severe data corruption and race conditions in the entity-component system.

## Data Pipeline
The primary function of this class is to translate a static data definition into a live change on an entity and a corresponding network message.

> Flow:
> Game Asset (JSON) -> Server `BuilderCodec` -> **ChangeStatInteraction** Instance -> `firstRun()` -> `CommandBuffer` -> `EntityStatMap` Component Update
>
> Parallel Flow:
> **ChangeStatInteraction** Instance -> `generatePacket()` / `configurePacket()` -> Network Queue -> Client Notification

