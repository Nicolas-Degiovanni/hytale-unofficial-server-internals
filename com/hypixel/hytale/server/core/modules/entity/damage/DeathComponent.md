---
description: Architectural reference for DeathComponent
---

# DeathComponent

**Package:** com.hypixel.hytale.server.core.modules.entity.damage
**Type:** Data Component

## Definition
```java
// Signature
public class DeathComponent implements Component<EntityStore> {
```

## Architecture & Concepts

The DeathComponent is a server-side, transient data component within the Entity-Component-System (ECS) architecture. It does not contain logic; instead, it serves as a data container that marks an entity as "dead" and stores the complete context of the death event.

Its primary role is to act as a message between systems. When a damage-handling system determines an entity has received fatal damage, it attaches a DeathComponent. Subsequent systems, such as those responsible for player respawning, item dropping, or broadcasting death messages, will then query for entities possessing this component to execute their logic.

The component is designed for serialization via its static CODEC field. This allows the state of a death event to be persisted in a world save or transmitted over the network, ensuring that clients can display accurate death screens and that the consequences of death are consistently applied. It decouples the moment of death from the consequences, allowing for a more modular and extensible death processing pipeline.

## Lifecycle & Ownership

-   **Creation:** A DeathComponent is never instantiated directly. It is created and attached to an entity exclusively through the static `tryAddComponent` helper methods. This is typically invoked by a core gameplay system (e.g., a `DamageSystem` or `HealthSystem`) at the exact moment an entity's health reaches zero or below. The initial state is seeded from a `Damage` object, which contains the immediate context of the fatal blow.

-   **Scope:** The component has an extremely short and well-defined lifecycle. It exists only for the few server ticks between an entity's death and its subsequent respawn or removal from the world. It is considered ephemeral state.

-   **Destruction:** The component is removed by a death-processing or entity-cleanup system once all consequences of the death have been applied. This includes dropping items, updating player state, and sending network packets. Failure to remove the component will result in the entity being incorrectly processed as "dead" in subsequent game ticks.

## Internal State & Concurrency

-   **State:** The DeathComponent is a mutable data object. Its fields are populated upon creation and may be further modified by systems within the death pipeline. For example, a system managing player statistics might update the component before a final death message is generated. It holds references to other data structures like `Message` and `InteractionChain`, but its core data, such as `deathCause`, is an asset ID string that requires resolution via the `AssetRegistry`.

-   **Thread Safety:** **This component is not thread-safe.** It is designed to be created, accessed, and modified exclusively on the main server game thread. All interactions with this component must be scheduled through the server's tick loop or via a `CommandBuffer`. Unsynchronized access from other threads will lead to data corruption and unpredictable server behavior. The presence of `tryAddComponent(CommandBuffer, ...)` is a strong indicator of this single-threaded design.

## API Surface

The public API is focused on providing access to the death context and facilitating its creation within the ECS framework.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getComponentType() | static ComponentType | O(1) | Retrieves the unique, engine-wide identifier for this component type. |
| tryAddComponent(store, ref, damage) | static void | O(1) | The primary factory method. Attaches a new DeathComponent to the entity if one is not already present. |
| getDeathCause() | DamageCause | O(log N) | Resolves the internal `deathCause` string ID into a full `DamageCause` asset via the `AssetRegistry`. May return null if the asset is not found. |
| getDeathItemLoss() | DeathItemLoss | O(1) | Constructs and returns a new value object encapsulating the rules for item loss on this specific death. |
| clone() | Component | O(1) | Creates a shallow copy of the component's data. |

## Integration Patterns

### Standard Usage

The component should only be added by a system that manages entity health and damage. The `CommandBuffer` variant of `tryAddComponent` is the preferred method, as it safely queues the component addition for the end of the current tick.

```java
// Inside a hypothetical DamageSystem processing a fatal damage event
public void processDamage(CommandBuffer<EntityStore> commands, Ref<EntityStore> target, Damage fatalDamage) {
    // This is the canonical way to mark an entity for death processing
    DeathComponent.tryAddComponent(commands, target, fatalDamage);
}
```

### Anti-Patterns (Do NOT do this)

-   **Direct Instantiation:** Do not call `new DeathComponent()`. The parameterless constructor exists solely for the serialization `CODEC`. The constructor taking a `Damage` object is protected and intended for internal use by the static factory methods. Bypassing `tryAddComponent` breaks the "add-if-not-present" safety check.

-   **Persistent State:** Do not allow this component to remain on an entity after it has respawned. It represents a single, transient event. Leaving it attached will cause death-processing systems to run repeatedly and incorrectly on a living entity.

-   **Manual Asset Resolution:** Do not manually store and look up the `deathCause` string. Always use the `getDeathCause()` method, which correctly interfaces with the `AssetRegistry`.

## Data Pipeline

The DeathComponent acts as a critical data payload in the server's death processing pipeline. Its flow is linear and event-driven.

> Flow:
> Fatal Damage Event -> Health/Damage System -> **DeathComponent.tryAddComponent** -> DeathComponent attached to Entity -> Death Processing Systems (Query for DeathComponent) -> Item Drop / Respawn Logic -> Component Removed from Entity

