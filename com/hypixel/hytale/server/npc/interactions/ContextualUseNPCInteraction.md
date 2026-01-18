---
description: Architectural reference for ContextualUseNPCInteraction
---

# ContextualUseNPCInteraction

**Package:** com.hypixel.hytale.server.npc.interactions
**Type:** Transient / Configuration Object

## Definition
```java
// Signature
public class ContextualUseNPCInteraction extends SimpleInstantInteraction {
```

## Architecture & Concepts
The **ContextualUseNPCInteraction** class represents a specific, server-side game action where a player interacts with an NPC, providing a predefined string of context. It is a concrete implementation of the broader interaction system, inheriting from **SimpleInstantInteraction**, which designates it as a single, non-continuous event.

This class acts as a data-driven command object. Its primary role is to be deserialized from game configuration files (e.g., JSON or HOCON defining an NPC's behavior) via its static **CODEC** field. This allows game designers to define specific NPC interactions and their contextual outcomes without writing Java code.

When triggered, its logic executes within the server's Entity Component System (ECS). It validates that the interactor is a **Player** and the target is an **NPCEntity**, then delegates the core logic to the target NPC's **Role** component. This class is the critical link between a configured interaction definition and the dynamic, stateful logic of an NPC entity in the game world.

### Lifecycle & Ownership
- **Creation:** Instances are not created manually using the constructor. They are instantiated by the Hytale serialization framework via the public static **CODEC** field when loading server-side game assets, typically as part of an NPC or item definition.
- **Scope:** An instance of this class is effectively a shared, immutable template for a specific type of interaction. It persists for the lifetime of the server session or until game assets are reloaded. It is not tied to a specific player or NPC instance but is reused for every execution of this configured action.
- **Destruction:** Managed by the asset loading system. Instances are eligible for garbage collection when the server shuts down or reloads its configuration assets.

## Internal State & Concurrency
- **State:** The object's state, consisting of its ID and the **context** string, is immutable after its creation during asset deserialization. The **firstRun** method does not modify the internal state of the **ContextualUseNPCInteraction** instance itself.
- **Thread Safety:** The object is inherently thread-safe for reading due to its immutable state. However, the execution of its primary method, **firstRun**, is **not** thread-safe and is designed to be called exclusively from the server's main game loop or a world-specific thread. It operates on a **CommandBuffer**, a standard ECS pattern for queueing world mutations to ensure they are applied sequentially and without race conditions.

## API Surface
The public contract is primarily for the interaction system and the serialization framework.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| CODEC | BuilderCodec | O(1) | Public static field used by the engine to serialize and deserialize this object from configuration files. |
| firstRun(type, context, cooldownHandler) | void | O(1) | **[Override]** The core logic entry point. Validates the interaction context and delegates the action to the target NPC's role component. Throws exceptions on invalid arguments. |

## Integration Patterns

### Standard Usage
This class is not intended to be used directly in Java code by most developers. Its primary integration point is through data files that define game behaviors. The engine's interaction module is responsible for finding the correct interaction object and invoking it.

The following example demonstrates how the *engine* would likely trigger this interaction.

```java
// Engine-level code (conceptual)
// An InteractionModule would execute this based on player input.
InteractionContext interactionCtx = buildContextForPlayerAction(player, targetNpc);
CooldownHandler cooldowns = getCooldownsForPlayer(player);

// The 'interaction' instance is retrieved from the target's configuration.
ContextualUseNPCInteraction interaction = targetNpc.getInteraction("USE_WITH_CONTEXT");

// The engine invokes the lifecycle method.
interaction.firstRun(InteractionType.USE, interactionCtx, cooldowns);
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not call `new ContextualUseNPCInteraction()`. The object will be incomplete and lack its configured context string. It must be created via the engine's asset loading and serialization pipeline.
- **Manual Invocation:** Do not call **firstRun** directly from custom game logic. This bypasses the server's core interaction system, including permission checks, cooldowns, and state management, which can lead to inconsistent game state or exploits.
- **State Modification:** Do not attempt to use reflection to modify the **context** field after initialization. These objects are shared across the server and such a change would affect all subsequent executions of this interaction.

## Data Pipeline
This class is a processor in the server-side player interaction pipeline. It translates a generic "use" event into a specific, context-aware command for an NPC.

> Flow:
> Player Input Packet -> Server Network Layer -> Interaction Module -> **ContextualUseNPCInteraction.firstRun()** -> CommandBuffer -> NPCEntity.Role.addContextualInteraction() -> Game State Update

---

