---
description: Architectural reference for ExitInstanceInteraction
---

# ExitInstanceInteraction

**Package:** com.hypixel.hytale.builtin.instances.interactions
**Type:** Data-Driven Command Object

## Definition
```java
// Signature
public class ExitInstanceInteraction extends SimpleInstantInteraction {
```

## Architecture & Concepts
The ExitInstanceInteraction class is a concrete implementation of the server-authoritative command pattern, designed to be configured and driven by game data rather than hard-coded logic. It represents a single, stateless action: ejecting an entity from a game *Instance* and returning it to a pre-defined location.

Architecturally, this class serves as a lightweight bridge between the generic server Interaction System and the specialized InstancesPlugin. It does not contain the complex logic for teleportation or instance management. Instead, its sole responsibility is to perform pre-condition checks and then delegate the core operation to the InstancesPlugin via a CommandBuffer.

The presence of a static CODEC field is a critical architectural indicator. It signifies that this class is intended to be deserialized from asset files (e.g., JSON or HOCON prefabs). This allows game designers to attach this "exit" behavior to any interactive entity, such as a portal or a pressure plate, without requiring engineering intervention.

The class inherits from SimpleInstantInteraction, which categorizes it as an action that resolves completely within a single server tick.

## Lifecycle & Ownership
- **Creation:** Instances of ExitInstanceInteraction are not created manually using the *new* keyword. They are instantiated by the engine's serialization framework via the provided CODEC when loading game assets, such as entity prefabs or world zone data.
- **Scope:** An instance of this class is effectively a stateless template. Its lifetime is tied to the asset that defines it. It persists as long as the parent asset is loaded in memory. It does not hold any per-interaction state.
- **Destruction:** The object is eligible for garbage collection when the associated asset sets are unloaded, for example, when a server shuts down or a world is no longer active.

## Internal State & Concurrency
- **State:** This class is **stateless and immutable**. It has no mutable instance fields. All necessary context for its execution (the interacting entity, the world state) is provided via the InteractionContext parameter in the firstRun method. This design ensures that a single shared instance can be used for all interactions of this type without side effects.

- **Thread Safety:** The object itself is thread-safe due to its immutability. However, the systems it interacts with, particularly the CommandBuffer, are **not thread-safe**.

    **WARNING:** All invocations of its methods must be performed on the primary server tick thread. Accessing it from other threads will lead to world state corruption, race conditions, and server crashes.

## API Surface
The public contract is minimal, focusing exclusively on the interaction execution hook.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| firstRun(type, context, cooldownHandler) | void | O(1) | Executes the interaction logic. Performs safety checks and delegates to the InstancesPlugin. |
| getWaitForDataFrom() | WaitForDataFrom | O(1) | Returns Server, indicating this is a fully server-authoritative interaction. |

## Integration Patterns

### Standard Usage
This class is not intended to be used directly in code. Instead, a game designer configures it within an entity's component data. The engine's interaction system is responsible for invoking it.

The conceptual flow for a developer is to rely on the system, not to call the class.

```java
// CONCEPTUAL: This code is managed by the engine's InteractionModule.
// A developer would not write this.

// 1. An entity with ExitInstanceInteraction is loaded from a prefab.
Entity portal = world.spawnEntity("my_portal_prefab");

// 2. A player interacts with the entity.
// 3. The engine finds the ExitInstanceInteraction component and executes it.
InteractionComponent interactionComp = portal.get(InteractionComponent.class);
ExitInstanceInteraction exitAction = interactionComp.getInteraction();

// 4. The engine provides the context and calls firstRun.
InteractionContext context = createInteractionContextFor(player, portal);
exitAction.firstRun(InteractionType.USE, context, cooldownHandler);
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not use `new ExitInstanceInteraction()`. The class is designed to be configured and managed by the asset loading system. Manual creation bypasses this entire data-driven architecture.
- **Manual Invocation:** Do not call the `firstRun` method directly. The Interaction System is responsible for creating the correct InteractionContext and managing the command buffer lifecycle. Bypassing the system can lead to an invalid or incomplete world state.
- **Stateful Subclassing:** Do not extend this class to add state. Its stateless nature is a core design assumption. Adding state will break thread safety guarantees and lead to unpredictable behavior when the engine reuses the interaction instance.

## Data Pipeline
This component acts as a command trigger within a larger data and control flow. It does not transform data but rather initiates a change in the entity's state.

> Flow:
> Player Input -> Network Packet -> Server Interaction Module -> **ExitInstanceInteraction.firstRun()** -> CommandBuffer.defer(InstancesPlugin.exitInstance) -> Entity Component System -> PendingTeleport component added to Player -> Teleportation System picks up component on next tick

