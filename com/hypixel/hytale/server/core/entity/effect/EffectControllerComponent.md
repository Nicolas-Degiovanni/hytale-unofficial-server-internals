---
description: Architectural reference for EffectControllerComponent
---

# EffectControllerComponent

**Package:** `com.hypixel.hytale.server.core.entity.effect`
**Type:** Component

## Definition
```java
// Signature
public class EffectControllerComponent implements Component<EntityStore> {
```

## Architecture & Concepts

The EffectControllerComponent is a server-side, data-oriented component within Hytale's Entity-Component-System (ECS) architecture. It serves as the definitive state machine for all temporary and permanent status effects applied to a single game entity. This component does not contain any active logic (like timers or update ticks); instead, it acts as a data container that is exclusively manipulated by server-side systems, primarily the **LivingEntityEffectSystem**.

Its core responsibilities include:
-   Tracking the identity, duration, and properties of all active effects on its parent entity.
-   Managing complex application logic based on an effect's **OverlapBehavior** (e.g., extending duration, ignoring subsequent applications, or overwriting existing effects).
-   Handling state changes that have visual or gameplay consequences, such as model swapping (transformations) or granting invulnerability.
-   Acting as a write-buffer for network synchronization. It meticulously records all state changes as a list of **EntityEffectUpdate** operations, which are consumed by the networking layer to inform clients.

This component is fundamental to character stats, combat mechanics, and visual feedback, bridging gameplay events with an entity's persistent state and its representation to players.

### Lifecycle & Ownership
-   **Creation:** An EffectControllerComponent is instantiated and attached to an entity by the ECS framework. This typically occurs when an entity is first created or when it is loaded from the **EntityStore**. The static **CODEC** field defines how this component is serialized and deserialized, making it a persistent part of an entity's data.
-   **Scope:** The lifecycle of this component is strictly bound to its parent entity. It persists as long as the entity exists in the world.
-   **Destruction:** The component is destroyed and garbage collected when its parent entity is removed from the world. The **clearEffects** method can be used to reset its state to default without destroying the component instance itself.

## Internal State & Concurrency
-   **State:** This component is highly mutable. Its primary state is held in the **activeEffects** map, which stores instances of **ActiveEntityEffect**. It also maintains a change log in the **changes** list and a dirty flag, **isNetworkOutdated**, to optimize network traffic. For performance, it maintains **cachedActiveEffectIndexes** to avoid repeated key set calculations. State related to model transformations is stored in **originalModel** and **activeModelChangeEntityEffectIndex**.

-   **Thread Safety:** **This component is not thread-safe.** It is designed to be accessed and modified exclusively by the main server tick thread. Any concurrent modification from other threads will lead to state corruption, race conditions, and unpredictable behavior. The use of non-concurrent collections from the fastutil library underscores this single-threaded design assumption.

## API Surface
The public API provides coarse-grained operations for managing the entire lifecycle of an effect. Direct manipulation of internal collections is disallowed.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| addEffect(...) | boolean | O(1) | Applies an entity effect. Manages overlap behavior. Returns false if the effect cannot be applied. |
| removeEffect(...) | void | O(1) | Removes an entity effect according to its specified RemovalBehavior. |
| clearEffects(...) | void | O(N) | Immediately removes all active effects from the entity. |
| consumeNetworkOutdated() | boolean | O(1) | Returns true if the component's state has changed since the last check, and resets the flag to false. |
| consumeChanges() | EntityEffectUpdate[] | O(N) | Returns an array of all state changes since the last call. Does not clear the internal change list. |
| clearChanges() | void | O(1) | Clears the internal list of pending network updates. |
| createInitUpdates() | EntityEffectUpdate[] | O(N) | Generates a complete snapshot of all active effects, used to initialize a new client's state. |

## Integration Patterns

### Standard Usage
This component should only be accessed and modified by server-side systems through a **ComponentAccessor**. The system is responsible for retrieving the component from an entity and calling its methods to apply or remove effects as a result of gameplay logic.

```java
// Executed within a server-side System
Ref<EntityStore> entityRef = ...;
ComponentAccessor<EntityStore> accessor = ...;

// Retrieve the component
EffectControllerComponent effects = accessor.getComponent(entityRef, EffectControllerComponent.getComponentType());

if (effects != null) {
    // Apply a new effect from an asset
    EntityEffect poisonEffect = EntityEffect.getAssetMap().getAsset("hytale:poison");
    effects.addEffect(entityRef, poisonEffect, accessor);
}
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Never create an instance using `new EffectControllerComponent()`. The ECS framework and EntityStore are solely responsible for the lifecycle of components. Direct instantiation will result in a disconnected object that is not part of any entity.
-   **State Tampering:** Do not retrieve the internal **activeEffects** map and modify it directly. Doing so will bypass critical logic, including network change tracking and stat recalculation triggers, leading to severe desynchronization.
-   **Cross-Thread Modification:** Do not access this component from any thread other than the world's primary update thread. This will cause crashes and data corruption.

## Data Pipeline
The EffectControllerComponent acts as a critical junction for effect-related data, transforming gameplay events into persistent state and network updates.

> Flow:
> Gameplay Event (e.g., Spell Hit) -> **LivingEntityEffectSystem** -> `addEffect()` call on **EffectControllerComponent** -> Internal state update (`activeEffects` map) -> Change is logged to `changes` list -> **Networking System** consumes changes via `consumeChanges()` -> **EntityEffectUpdate** packet sent to client -> Client-side visual and UI updates.

