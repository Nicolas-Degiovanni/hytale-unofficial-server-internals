---
description: Architectural reference for ChangeStatWithModifierInteraction
---

# ChangeStatWithModifierInteraction

**Package:** com.hypixel.hytale.server.core.modules.interaction.interaction.config.server
**Type:** Transient / Configuration Object

## Definition
```java
// Signature
public class ChangeStatWithModifierInteraction extends ChangeStatBaseInteraction {
```

## Architecture & Concepts
The ChangeStatWithModifierInteraction class is a server-side, data-driven component responsible for executing a specific type of entity stat modification. It extends the functionality of its parent, ChangeStatBaseInteraction, by introducing a dynamic calculation step that incorporates modifiers from an entity's equipped armor.

This class embodies the engine's "Interaction System" design, where complex game mechanics are defined as composable, deserializable objects rather than hard-coded logic. It acts as a command object, encapsulating the logic for a single, configurable action—in this case, "change a stat, but first check the target's armor for relevant buffs or debuffs".

Its primary role is to bridge asset configuration with the live entity-component system. An instance of this class is not a service but a template for an effect, loaded from game files. When an in-game event triggers this interaction, its logic is executed against a target entity within a specific transaction-like context provided by the CommandBuffer.

The static CODEC field is the most critical architectural feature, indicating that this class is intended to be instantiated by the engine's serialization system from asset files (e.g., JSON definitions for items or abilities). This makes the behavior extensible and moddable without recompiling the server.

## Lifecycle & Ownership
-   **Creation:** Instances are not created directly using the constructor in game logic. They are deserialized from asset configuration files by the engine's `BuilderCodec` system during server startup or when asset packs are loaded. The static `CODEC` field serves as the factory and deserializer.
-   **Scope:** The object's lifetime is tied to the asset definition it is part of, such as an Item configuration. It is effectively a stateless template that persists as long as the parent asset is loaded in memory.
-   **Destruction:** The object is marked for garbage collection when the server unloads the associated assets, typically during a server shutdown or world change.

## Internal State & Concurrency
-   **State:** Instances of this class are effectively immutable after deserialization. The core configuration fields, `interactionModifierId` and the inherited `entityStats` map, are populated by the CODEC and are not intended to be modified at runtime. The `firstRun` method is purely computational; it reads this configuration and the target entity's state to produce a result, but does not mutate its own state.
-   **Thread Safety:** This class is inherently thread-safe due to its immutable state. The `firstRun` method is designed to be executed within a single-threaded context for a given entity, typically the main server tick or a dedicated entity worker thread. It relies on the calling system to provide exclusive, safe access to the entity's components via the `InteractionContext` and `CommandBuffer`. No internal locking mechanisms are present or required.

## API Surface
The public contract is minimal, exposing only the logic execution entry point.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| firstRun(type, context, cooldownHandler) | void | O(S * A) | Executes the stat modification logic. Complexity is determined by the number of stats to change (S) multiplied by the number of armor slots (A) on the target entity. Throws NullPointerException if context is invalid. |

## Integration Patterns

### Standard Usage
This class is not intended for direct invocation by module developers. It is configured declaratively in asset files and executed automatically by the server's core Interaction Module. The following example is a conceptual illustration of how the system might invoke its logic.

```java
// Conceptual example of engine-level invocation
// A developer would NOT write this code.

// 1. An interaction is triggered (e.g., player uses an item).
// 2. The engine retrieves the configured interaction object.
ChangeStatWithModifierInteraction interaction = item.getInteraction();

// 3. The engine prepares the context for the target entity.
InteractionContext context = createInteractionContextFor(targetEntity);
CooldownHandler cooldowns = getCooldownHandlerFor(sourceEntity);

// 4. The interaction logic is executed.
interaction.firstRun(InteractionType.PRIMARY, context, cooldowns);
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Never use `new ChangeStatWithModifierInteraction()`. The object is not designed for manual construction and will be in an invalid state. It must be created via the `CODEC` deserializer from a valid asset file.
-   **State Mutation:** Do not attempt to modify the `interactionModifierId` or other fields after the object has been loaded. This violates its immutable design and can lead to unpredictable behavior across the server.
-   **External Invocation:** Calling `firstRun` outside of the engine's managed Interaction System loop is dangerous. The method requires a valid `InteractionContext` and `CommandBuffer` to guarantee transactional and thread-safe access to entity state.

## Data Pipeline
The flow of data and logic for this component is sequential and transactional, managed by the server's entity processing loop.

> Flow:
> Player Input (Use Item) -> Server Network Handler -> Interaction Module -> **ChangeStatWithModifierInteraction.firstRun()**
>
> **Inside firstRun:**
> 1.  Reads base stat changes from its own configuration (`this.entityStats`).
> 2.  Acquires target `EntityStore` and `CommandBuffer` from the `InteractionContext`.
> 3.  Fetches the target's `Inventory` component.
> 4.  Iterates through equipped armor `ItemStack`s.
> 5.  For each armor piece, reads its `StaticModifier` map.
> 6.  Calculates the total additive and multiplicative modifiers for each stat.
> 7.  Applies the calculated modifiers to the base stat changes.
> 8.  Submits the final stat changes to the `EntityStatMap` component via the `CommandBuffer`.
>
> **Post-Execution:**
> CommandBuffer Commit -> Entity State Updated -> State Replicated to Client -> Client UI Update (e.g., Health Bar)

