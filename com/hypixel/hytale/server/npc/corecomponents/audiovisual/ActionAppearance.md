---
description: Architectural reference for ActionAppearance
---

# ActionAppearance

**Package:** com.hypixel.hytale.server.npc.corecomponents.audiovisual
**Type:** Transient

## Definition
```java
// Signature
public class ActionAppearance extends ActionBase {
```

## Architecture & Concepts
The ActionAppearance class is a concrete command object within the server-side NPC behavioral framework. It represents a single, atomic action that an NPC can perform: changing its visual appearance. This is not a long-lived service but a transient, data-driven object that encapsulates both the *what* (the target appearance string) and the *how* (the logic to apply it to an entity).

This class operates as a node within a larger AI structure, such as a Behavior Tree or a Finite State Machine, managed by an NPC's Role. Its design strictly adheres to an Entity-Component-System (ECS) paradigm. It does not hold state about the NPC it is acting upon; instead, all necessary world state, including the entity reference (Ref) and the component data store (Store), is passed into its methods during the AI update tick.

## Lifecycle & Ownership
- **Creation:** ActionAppearance is instantiated exclusively by its corresponding builder, BuilderActionAppearance. This typically occurs once at server startup or during zone loading when NPC behavior definitions are parsed from configuration files. The resulting object is part of an immutable behavior graph.
- **Scope:** The object is stateless and reusable. A single ActionAppearance instance can be shared by all NPCs that use the same behavior definition. Its lifetime is tied to the lifetime of the NPC behavior asset in memory.
- **Destruction:** The object is eligible for garbage collection only when the server unloads the corresponding NPC behavior definitions, for instance, during a full server shutdown or a hot-reload of game assets.

## Internal State & Concurrency
- **State:** Immutable. The core state, the *appearance* string, is a final field set during construction. The object itself does not cache data or maintain any state related to a specific NPC entity.
- **Thread Safety:** This class is inherently thread-safe due to its immutable nature. However, the execution context in which it operates is not. The Store parameter passed to its methods represents a mutable view of the game world.

    **WARNING:** The calling system, typically the NPC's Role or AI scheduler, is responsible for ensuring that all actions for a given entity or world region are executed on a single, designated thread. Concurrent modification of the EntityStore via this or any other action will lead to race conditions, data corruption, and server instability.

## API Surface
The public contract is defined by its parent, ActionBase, and consists of two primary methods for use by the AI scheduler.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| canExecute(...) | boolean | O(1) | Checks if the action can be performed. Returns false if the configured appearance string is null or empty. |
| execute(...) | boolean | O(1) | Applies the appearance change to the target NPC entity via the EntityStore. Asserts that the NPCEntity component exists. |

## Integration Patterns

### Standard Usage
ActionAppearance is not intended to be invoked directly by general game logic. It is designed to be held and executed by a higher-level AI component, such as a Role, as part of an NPC's update cycle.

```java
// Within a hypothetical AI scheduler or Role update method
ActionAppearance changeLook = npcBehavior.getAction(); // Retrieve the pre-configured action

// The context (ref, role, etc.) is provided by the scheduler's tick
if (changeLook.canExecute(ref, role, sensorInfo, dt, store)) {
    changeLook.execute(ref, role, sensorInfo, dt, store);
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never use `new ActionAppearance()`. The object must be constructed via its designated builder, BuilderActionAppearance, to ensure its state is correctly initialized from the game's data files.
- **Stateful Logic:** Do not modify this class to hold state related to a specific NPC. All contextual data must flow through the method parameters.
- **Skipping Preconditions:** Do not call `execute` without a preceding successful call to `canExecute`. While the current implementation is simple, this violates the component's contract and may bypass critical validation logic in the future.

## Data Pipeline
The primary function of this class is to inject a new value into the server's Entity-Component-System data store. This change is then propagated to clients via the standard network synchronization pipeline.

> Flow:
> NPC AI Scheduler -> **ActionAppearance.execute()** -> NPCEntity.setAppearance() -> EntityStore Write -> Network Synchronization System -> Packet Sent to Client -> Client-side Model Update

