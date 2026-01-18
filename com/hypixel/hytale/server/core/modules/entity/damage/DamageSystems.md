---
description: Architectural reference for DamageSystems
---

# DamageSystems

**Package:** com.hypixel.hytale.server.core.modules.entity.damage
**Type:** Utility

## Definition
```java
// Signature
public class DamageSystems {
```

## Architecture & Concepts

The DamageSystems class is a static container for the suite of Entity Component Systems (ECS) that collectively manage the entire lifecycle of damage application on the server. It is not a single system but rather the central orchestration point for a multi-stage damage pipeline. This class embodies the "Systems" aspect of ECS for all combat and environmental damage.

The core architectural pattern is a phased pipeline, controlled by System Groups within the ECS scheduler. Damage is not applied instantaneously. Instead, a damage event is initiated and then passed through a series of systems, each responsible for a specific task:

1.  **Gathering:** Systems that generate damage from game state (e.g., fall damage, drowning).
2.  **Filtering:** Systems that apply reductions, immunities, or cancellations (e.g., armor, invulnerability, PvP rules).
3.  **Application:** The core system that subtracts health from the entity's stats.
4.  **Inspection:** A final set of systems that trigger side effects based on the applied damage (e.g., particles, sounds, animations, network updates).

This decoupled, multi-stage approach allows for complex and extensible combat logic without creating monolithic, unmanageable classes. The static `executeDamage` methods serve as the sole entry point into this pipeline for other parts of the engine.

## Lifecycle & Ownership

-   **Creation:** The DamageSystems class itself is a static utility and is never instantiated. The nested system classes (e.g., ApplyDamage, FallDamagePlayers) are instantiated once by the DamageModule during server bootstrap. They are registered with the ECS world scheduler according to their specified dependencies and system groups.
-   **Scope:** The instantiated damage systems persist for the entire server session. They are fundamental components of the core game loop.
-   **Destruction:** The systems are discarded and garbage collected only when the server shuts down and the ECS world is torn down.

## Internal State & Concurrency

-   **State:** The DamageSystems class and its nested systems are fundamentally stateless. All state they operate on is external, residing in entity components (like EntityStatMap, TransformComponent) or within the transient Damage event object passed through the pipeline.
-   **Thread Safety:** These systems are designed to be executed by the Hytale ECS scheduler, which may run them in parallel across different entity archetypes. Thread safety is achieved through several mechanisms:
    -   **Command Buffers:** All structural mutations (adding/removing components) are deferred via a CommandBuffer, which executes these changes in a synchronized step between system updates.
    -   **Statelessness:** Since systems are stateless, multiple instances can operate on different data sets without interfering with each other.
    -   **Explicit Parallelism:** Some systems include an `isParallel` method to explicitly declare their suitability for parallel execution.

**WARNING:** Directly invoking the `tick` or `handle` methods on these systems from outside the ECS scheduler is not thread-safe and will lead to data corruption and server instability.

## API Surface

The primary public contract consists of the static `executeDamage` methods, which are the designated entry points for initiating a damage event.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| executeDamage(Ref, ComponentAccessor, Damage) | static void | O(1) | Submits a damage event for an entity via a ComponentAccessor. |
| executeDamage(int, ArchetypeChunk, CommandBuffer, Damage) | static void | O(1) | Submits a damage event for an entity within a chunk via a CommandBuffer. |
| executeDamage(Ref, CommandBuffer, Damage) | static void | O(1) | The most common variant. Submits a damage event for an entity reference via a CommandBuffer. |

## Integration Patterns

### Standard Usage

All game logic that needs to inflict damage—such as weapon interactions, spell effects, or environmental hazards—must do so by creating a Damage object and submitting it to the pipeline using `executeDamage`.

```java
// Example: Applying damage from a projectile hit
Damage damage = new Damage(new Damage.EntitySource(projectileAttackerRef), DamageCause.PROJECTILE, 25.0F);

// Use the CommandBuffer to invoke the damage event on the target entity
DamageSystems.executeDamage(targetEntityRef, commandBuffer, damage);
```

### Anti-Patterns (Do NOT do this)

-   **Direct Instantiation:** Do not use `new ApplyDamage()`. The systems are managed exclusively by the DamageModule and the ECS scheduler.
-   **Manual Invocation:** Never call the `handle` or `tick` methods directly. The scheduler is responsible for executing systems in the correct order based on their declared dependencies. Bypassing the scheduler will break the damage pipeline and cause unpredictable behavior.
-   **Stateful Systems:** Do not add mutable instance fields to system classes. This violates the stateless principle of ECS and will cause severe issues in a parallel execution environment.

## Data Pipeline

The flow of a single damage event is a clear, multi-stage pipeline orchestrated by the ECS scheduler using System Groups. A `Damage` object is created and passed sequentially through systems registered in each group.

> Flow:
> **1. Event Generation (GatherDamageGroup)**
> Systems like `FallDamagePlayers`, `CanBreathe`, and `OutOfWorldDamage` monitor entity state. When conditions are met, they create a `Damage` object and inject it into the pipeline for a specific entity using `executeDamage`.
>
> **2. Filtering & Modification (FilterDamageGroup)**
> The `Damage` object is processed by systems like `ArmorDamageReduction`, `FilterUnkillable`, and `PlayerDamageFilterSystem`. These systems can reduce the damage amount, apply multipliers, or cancel the event entirely by calling `damage.setCancelled(true)`.
>
> **3. Health Application**
> The `ApplyDamage` system executes. If the event has not been cancelled, it reads the final damage amount and subtracts it from the target's health stat via the `EntityStatMap` component. If health reaches zero, it adds a `DeathComponent` to the entity.
>
> **4. Post-Damage Side Effects (InspectDamageGroup)**
> A final group of systems like `ApplyParticles`, `ApplySoundEffects`, `HitAnimation`, and `PlayerHitIndicators` react to the successful application of damage. They read the final `Damage` object to trigger visual effects, audio cues, animations, and send necessary network packets to clients.

