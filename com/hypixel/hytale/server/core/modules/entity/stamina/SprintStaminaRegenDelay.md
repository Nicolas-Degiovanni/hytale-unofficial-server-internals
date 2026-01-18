---
description: Architectural reference for SprintStaminaRegenDelay
---

# SprintStaminaRegenDelay

**Package:** com.hypixel.hytale.server.core.modules.entity.stamina
**Type:** Component / Data Model

## Definition
```java
// Signature
public class SprintStaminaRegenDelay implements Resource<EntityStore> {
```

## Architecture & Concepts
The SprintStaminaRegenDelay class is a data component that represents a temporary state applied to a game entity. Its primary function is to model the delay that occurs after an entity stops sprinting before its stamina begins to regenerate. It is not a self-contained system; rather, it is a piece of state managed by the broader StaminaModule.

This class implements the Resource interface, indicating it is a component managed within an Entity-Component-System (ECS) or a similar architectural pattern, where the EntityStore serves as the component container for an entity.

A key architectural feature is its built-in cache invalidation mechanism. It uses a static AtomicInteger, ASSET_VALIDATION_STATE, as a global "version" number for all resources of this type. Each instance holds its own local validationState. When game data is reloaded or global stats change, the static invalidateResources method is called, incrementing the global version. This instantly marks all existing instances as stale, as their local version no longer matches the global one. This is a highly efficient method for triggering mass re-computation of entity stats without iterating through every entity in the world.

## Lifecycle & Ownership
-   **Creation:** Instances are not meant to be created directly by developers. The StaminaModule is responsible for creating and attaching this component to an entity when an action, such as sprinting, incurs a regeneration delay. The presence of a copy constructor and a clone method suggests that instances are often created by duplicating a template or a previous state.
-   **Scope:** The lifecycle of a SprintStaminaRegenDelay instance is tightly coupled to the entity it is attached to. It persists as a component within the entity's EntityStore for as long as the regeneration delay is active.
-   **Destruction:** The component is effectively deactivated via the markEmpty method, which resets its state. The Java object itself is subject to garbage collection when the parent entity is destroyed or when the StaminaModule explicitly removes the component from the EntityStore.

## Internal State & Concurrency
-   **State:** This object is highly mutable. Its core fields, statIndex and statValue, are frequently modified by the managing system (StaminaModule) to count down the delay. It is fundamentally a stateful data container.
-   **Thread Safety:** This class is **not thread-safe**. All instance fields are accessed and modified without any synchronization primitives. The static ASSET_VALIDATION_STATE is an AtomicInteger, making the invalidation trigger itself safe. However, all interactions with an instance of this class, such as calling update or reading getValue, must be performed from a single, synchronized context, typically the main server game loop. Unmanaged multi-threaded access will lead to race conditions and corrupted entity state.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getResourceType() | ResourceType | O(1) | Static method to retrieve the unique type identifier for this component. |
| validate() | boolean | O(1) | Checks if the instance's data is current against the global asset version. |
| hasDelay() | boolean | O(1) | Returns true if a regeneration delay is currently active. |
| update(int, float) | void | O(1) | Sets the delay state. This also marks the component as valid. |
| markEmpty() | void | O(1) | Resets the component to a non-delay state. |
| invalidateResources() | void | O(1) | Static method to globally invalidate all instances of this component. |

## Integration Patterns

### Standard Usage
This component should never be instantiated or managed directly. It is retrieved from an entity's component storage and manipulated by a governing system, such as the StaminaModule.

```java
// Example within a hypothetical StaminaSystem update loop
EntityStore entityStore = entity.getStore();
SprintStaminaRegenDelay delayComponent = entityStore.getResource(SprintStaminaRegenDelay.getResourceType());

if (delayComponent != null && !delayComponent.validate()) {
    // Data is stale, recalculate from base stats and update
    recalculateStaminaDelay(entity, delayComponent);
}

if (delayComponent != null && delayComponent.hasDelay()) {
    // Tick the delay timer down
    float newValue = delayComponent.getValue() + timeDelta;
    delayComponent.update(delayComponent.getIndex(), newValue);
}
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Do not use `new SprintStaminaRegenDelay()`. The component's lifecycle is managed by the ECS framework and the StaminaModule. Direct creation bypasses the managing system and will not be registered with the entity.
-   **State Polling:** Do not repeatedly call `getValue()` in a tight loop. The value only changes once per game tick as updated by its managing system.
-   **Asynchronous Modification:** Do not modify the component from a separate thread. All reads and writes must be synchronized with the main game thread to prevent data corruption.

## Data Pipeline
SprintStaminaRegenDelay acts as a stateful node in the entity data flow, not a processing pipeline itself.

> **Delay Application Flow:**
> Player Action (e.g., Sprint) -> StaminaModule -> **SprintStaminaRegenDelay.update()** (Sets initial delay) -> Component attached to EntityStore

> **Delay Countdown Flow:**
> Game Tick -> StaminaModule -> Reads **SprintStaminaRegenDelay** from EntityStore -> Updates internal timer -> **SprintStaminaRegenDelay.update()** (Writes new value)

> **Invalidation Flow:**
> Global Event (e.g., Asset Reload) -> **SprintStaminaRegenDelay.invalidateResources()** -> Next Game Tick -> StaminaModule calls **validate()** (returns false) -> System recalculates stats and calls **update()**

