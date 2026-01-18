---
description: Architectural reference for SprintStaminaEffectSystem
---

# SprintStaminaEffectSystem

**Package:** com.hypixel.hytale.server.core.modules.entity.stamina
**Type:** System Component

## Definition
```java
// Signature
public static class SprintStaminaEffectSystem extends EntityTickingSystem<EntityStore> {
```

## Architecture & Concepts
The SprintStaminaEffectSystem is a server-side system within the Entity Component System (ECS) framework. Its sole responsibility is to enforce a delay on stamina regeneration immediately after a player stops sprinting. It functions as a state controller, intervening in the game loop to temporarily modify an entity's stats.

This system operates on a specific subset of entities defined by its internal query: those that possess a Player component, an EntityStatMap, and a MovementStatesComponent.

Architecturally, its position in the execution order is critical. It is explicitly scheduled to run **before** the primary `PlayerRegenerateStatsSystem`. By running first, it can preemptively set a "regeneration delay" stat on the player's EntityStatMap. When the subsequent `PlayerRegenerateStatsSystem` executes, it reads this stat and bypasses its normal stamina regeneration logic, effectively creating the desired delay. This demonstrates a "state-setter" pattern within a multi-stage, deterministic game loop.

The system's configuration, such as the duration of the delay, is not hardcoded. It is dynamically loaded from the world's `GameplayConfig`, making it tunable by server administrators or game designers without requiring code changes.

## Lifecycle & Ownership
- **Creation:** Instantiated automatically by the server's ECS framework when the `StaminaSystems` module is loaded during world initialization. It is never created manually.
- **Scope:** The instance is scoped to a single `EntityStore` and persists for the entire lifecycle of a server world.
- **Destruction:** The instance is marked for garbage collection when the server world is shut down and the parent ECS framework is dismantled.

## Internal State & Concurrency
- **State:** This class is fundamentally **stateless**. Its fields are immutable configurations (component type handles, queries, dependency lists) that do not change after construction. It does not cache any per-entity or per-tick data. All state it reads and modifies is external, located in ECS Components and Resources.

- **Thread Safety:** This system is **not thread-safe** and is designed to be executed exclusively by the main server thread that drives the ECS game loop. The ECS scheduler guarantees that its `tick` method is called in a serialized, deterministic order relative to other systems. Any attempt to access this system from another thread will result in data corruption and catastrophic state desynchronization.

## API Surface
The public methods of this class are framework overrides intended for the ECS scheduler, not for direct invocation by developers.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| tick(dt, store) | void | O(N) | Framework entry point. Iterates over all entities matching the query. N is the number of matched entities. |
| getQuery() | Query | O(1) | Framework callback to retrieve the entity selection criteria. |
| getDependencies() | Set | O(1) | Framework callback to declare execution order relative to other systems. |

## Integration Patterns

### Standard Usage
A developer does not interact with this class directly. Its behavior is enabled by including its parent module in the server configuration and is configured through the `StaminaGameplayConfig` asset. The system is entirely managed by the ECS framework.

To influence its behavior, a game designer would modify the gameplay configuration file:

```yaml
# Example GameplayConfig snippet
plugins:
  StaminaGameplayConfig:
    sprintRegenDelay:
      index: 12 # The entity stat index for stamina regen delay
      value: 60 # The delay value in ticks (e.g., 3 seconds at 20 TPS)
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never call `new SprintStaminaEffectSystem()`. The ECS framework is responsible for its creation and injection into the system graph. Manual creation will result in a non-functional object that is ignored by the game loop.
- **Manual Invocation:** Do not call the `tick` method directly. This bypasses the dependency scheduler, query execution, and command buffer synchronization, which will break game state determinism and likely cause crashes.
- **Configuration via Code:** Do not attempt to modify the system to read configuration from other sources. The `updateResource` method is designed to be the single point of truth for loading configuration from the `GameplayConfig`.

## Data Pipeline
The system participates in two distinct data flows: a one-time configuration flow and a per-tick entity processing flow.

**Configuration Flow (On-demand):**
> GameplayConfig File -> `StaminaGameplayConfig` Asset -> **SprintStaminaEffectSystem.updateResource** -> `SprintStaminaRegenDelay` ECS Resource

**Per-Tick Processing Flow:**
> Player Input (Stops Sprinting) -> `MovementStatesComponent` is updated -> **SprintStaminaEffectSystem.tick** (Reads `MovementStatesComponent`, Writes to `EntityStatMap`) -> `PlayerRegenerateStatsSystem` (Reads modified `EntityStatMap` and pauses regeneration)

