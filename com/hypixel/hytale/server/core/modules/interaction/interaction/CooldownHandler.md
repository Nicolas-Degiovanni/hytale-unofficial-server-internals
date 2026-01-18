---
description: Architectural reference for CooldownHandler
---

# CooldownHandler

**Package:** com.hypixel.hytale.server.core.modules.interaction.interaction
**Type:** Managed Component

## Definition
```java
// Signature
public final class CooldownHandler implements Tickable {
```

## Architecture & Concepts

The CooldownHandler is a server-side system responsible for managing time-based restrictions on actions, commonly known as cooldowns. It acts as a centralized registry and state machine for all cooldowns within its operational scope, such as a player's abilities or an entity's behaviors.

Architecturally, this class is a core component of the gameplay logic loop. By implementing the **Tickable** interface, it integrates directly with the server's main game thread, receiving periodic updates (ticks) that allow it to advance the state of all active cooldowns over time. This design ensures that all time-based calculations are synchronized with the server's simulation rate.

The handler manages a collection of inner **Cooldown** objects, each representing a distinct, stateful timer. These objects are stored in a map keyed by a unique String identifier, allowing game logic to query or modify specific cooldowns by name. This model supports complex cooldown mechanics, including multi-charge systems where an action can be used multiple times before a longer recharge period is required.

## Lifecycle & Ownership

-   **Creation:** The CooldownHandler is not intended for direct instantiation. It is created and managed by a higher-level service or module manager, typically during the initialization of the server's interaction module.
-   **Scope:** An instance of CooldownHandler is long-lived, persisting for the entire duration of the game session or world it is associated with. Its internal state (the map of active cooldowns) is dynamic.
-   **Destruction:** The handler is destroyed when its parent module is shut down, for example, during a server stop or world unload. Individual Cooldown objects are automatically removed from the internal map by the `tick` method once they have fully recharged and are no longer active, preventing memory leaks from expired timers.

## Internal State & Concurrency

-   **State:** This class is highly stateful and mutable. Its primary state is the `cooldowns` map, which holds all active Cooldown instances. Each Cooldown object maintains its own mutable state, including remaining time, charge count, and recharge progress.
-   **Thread Safety:** The class is designed for a specific concurrency model. The top-level map is a **ConcurrentHashMap**, making lookups, additions, and removals of Cooldown objects thread-safe. This allows different threads (e.g., network packet processors) to safely query the cooldown status of an action.

    **WARNING:** While the map itself is thread-safe, the inner Cooldown objects are **not**. All state modification, particularly the time-advancement logic in the `tick` method, is expected to occur on a single, dedicated game loop thread. Concurrent calls to methods that mutate a Cooldown object's internal timers from multiple threads will lead to race conditions and unpredictable behavior.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| isOnCooldown(root, id, ...) | boolean | O(1) | Checks if a specific action is on cooldown. Lazily creates a new Cooldown state if one does not exist and `force` is true. |
| resetCooldown(id, ...) | void | O(1) | Forces a specific cooldown to its maximum duration and resets its charges. |
| getCooldown(id) | Cooldown | O(1) | Retrieves the state object for a given cooldown ID, or null if it does not exist. |
| tick(dt) | void | O(N) | The primary update method. Iterates all active cooldowns (N) and advances their timers by the delta time (dt). |

## Integration Patterns

### Standard Usage

The CooldownHandler should be retrieved from a central service registry or context. Game logic then uses it as a gatekeeper before executing a restricted action.

```java
// In a game logic class, e.g., an ability handler
CooldownHandler cooldowns = context.getService(CooldownHandler.class);
String abilityId = "player.dash";

// Before executing the dash ability
if (!cooldowns.isOnCooldown(rootInteraction, abilityId, ...)) {
    // Execute the ability
    // The isOnCooldown call itself will trigger the cooldown
} else {
    // Deny the action, player must wait
}
```

### Anti-Patterns (Do NOT do this)

-   **Direct Instantiation:** Never call `new CooldownHandler()`. This creates an orphaned instance that is not ticked by the game loop, meaning its cooldowns will never expire. Always retrieve the managed instance from the engine's service context.
-   **External State Mutation:** Avoid retrieving a Cooldown object and manually calling its mutator methods like `increaseTime` or `deductCharge` unless you have a deep understanding of the gameplay system. Prefer using the higher-level CooldownHandler API to ensure consistent behavior.
-   **Ticking from Multiple Threads:** The `tick` method must only be called from the main server game loop. Calling it from other threads will break the concurrency model and corrupt cooldown state.

## Data Pipeline

The CooldownHandler functions as a stateful gate in a control flow rather than a data processing pipeline.

> **Action Request Flow:**
> Player Input -> Server Command -> **Action Logic** -> `isOnCooldown(id)` -> **CooldownHandler** -> Returns *false* -> Action Logic Proceeds

> **Time Progression Flow:**
> Server Game Loop -> `tick(dt)` -> **CooldownHandler.tick(dt)** -> Iterates all `Cooldown.tick(dt)` -> Internal Timers Updated

