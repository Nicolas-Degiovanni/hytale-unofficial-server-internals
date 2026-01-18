---
description: Architectural reference for AvoidanceSystem
---

# AvoidanceSystem

**Package:** com.hypixel.hytale.server.npc.systems
**Type:** System Component

## Definition
```java
// Signature
public class AvoidanceSystem extends SteppableTickingSystem {
```

## Architecture & Concepts

The AvoidanceSystem is a core component of the server-side NPC AI framework, operating within the Entity Component System (ECS) architecture. Its primary function is to provide localized, dynamic collision avoidance and group separation behaviors for NPCs.

This system acts as a post-processor for NPC movement intentions. It runs *after* the primary `BehaviourTickSystem`, which determines an NPC's high-level goal and initial steering direction. The AvoidanceSystem then refines this steering vector by blending in forces that push the NPC away from nearby obstacles and other entities.

It orchestrates two distinct but related behaviors:
1.  **Avoidance:** Calculating steering adjustments to prevent collisions with designated entities or environmental hazards.
2.  **Separation:** A flocking-style behavior that encourages NPCs to maintain a minimum distance from their peers, preventing clumping.

Crucially, the system itself is a lightweight orchestrator. It queries for eligible NPCs and delegates the complex mathematical calculations to the `Role` component attached to each `NPCEntity`. This design keeps the system focused on iteration and scheduling, while the stateful logic resides within the components themselves. The system also includes extensive hooks for debug visualization, rendering steering vectors directly in the world to aid designers and engineers.

## Lifecycle & Ownership

-   **Creation:** Instantiated once by the server's central ECS System Registry during world initialization. It is not intended for manual creation.
-   **Scope:** The singleton instance persists for the entire lifetime of the server world. As a stateless system, its instance is reused across all ticks.
-   **Destruction:** The instance is discarded and eligible for garbage collection when the server world is unloaded or during a full server shutdown.

## Internal State & Concurrency

-   **State:** The AvoidanceSystem is fundamentally stateless. Its fields, such as `query` and `dependencies`, are immutable references configured at construction. All mutable state related to avoidance, such as steering vectors or ignored entities, is stored within the `Role` component of each individual NPC.
-   **Thread Safety:** This system is designed for high-performance, parallel execution. The `isParallel` method allows the ECS scheduler to process different chunks of entities across multiple threads simultaneously. Thread safety is achieved through several ECS patterns:
    -   Each invocation of `steppedTick` operates on a unique entity, preventing data races on component data.
    -   World-state modifications and entity queries are performed through a `CommandBuffer`, which safely defers and synchronizes operations, ensuring that reads and writes do not conflict between threads.

## API Surface

The public API is exclusively for consumption by the ECS scheduler. Developers interact with this system indirectly by configuring components.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| steppedTick(dt, index, chunk, store, buffer) | void | O(N) | The main entry point called by the ECS scheduler for each entity matching the query. N is the number of nearby entities considered for avoidance. This method should never be called directly. |
| getDependencies() | Set | O(1) | Defines execution order relative to other systems. Ensures it runs after `BehaviourTickSystem`. |
| getQuery() | Query | O(1) | Defines the component signature for entities this system will process (`NPCEntity` and `TransformComponent`). |

## Integration Patterns

### Standard Usage

Developers do not interact with the AvoidanceSystem class directly. Its behavior is enabled and configured on a per-entity basis through the `Role` component.

```java
// Example: Enabling avoidance and separation on an NPC's Role
// This code would exist within an NPC's behavior or initialization logic.

NPCEntity npc = ...; // Get the NPC component
Role role = npc.getRole();

// Enable the behaviors processed by AvoidanceSystem
role.setAvoidingEntities(true);
role.setApplySeparation(true);

// The ECS scheduler will automatically run AvoidanceSystem on this entity
// because it matches the query and the flags are enabled.
```

### Anti-Patterns (Do NOT do this)

-   **Direct Instantiation:** Never call `new AvoidanceSystem()`. The system's lifecycle is managed entirely by the ECS framework. Manual instantiation will result in a non-functional object that is not registered with the game loop.
-   **Manual Invocation:** Do not call `steppedTick` directly. This bypasses dependency ordering, parallel execution, and the CommandBuffer, which will lead to race conditions, inconsistent world state, and crashes.
-   **Stateful Logic:** Do not add mutable fields to this class. Systems must remain stateless to support parallel execution and predictable behavior.

## Data Pipeline

The system acts as a filter or modifier in the NPC motion data pipeline. It consumes an initial steering vector and outputs a refined one, which is then used by downstream physics systems.

> Flow:
> `BehaviourTickSystem` (calculates goal-oriented steering) -> `Role.bodySteering` (stores initial vector) -> **AvoidanceSystem** (reads `bodySteering`, queries world, blends avoidance forces) -> `Role.bodySteering` (updated with final vector) -> Motion System (applies final steering to update `TransformComponent`)

