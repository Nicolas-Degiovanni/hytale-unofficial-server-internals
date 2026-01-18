---
description: Architectural reference for KnockbackSystems
---

# KnockbackSystems

**Package:** com.hypixel.hytale.server.core.entity.knockback
**Type:** Utility

## Definition
```java
// Signature
public class KnockbackSystems {
    public static class ApplyKnockback extends EntityTickingSystem<EntityStore> implements IVelocityModifyingSystem {
        // ...
    }

    public static class ApplyPlayerKnockback extends EntityTickingSystem<EntityStore> implements IVelocityModifyingSystem {
        // ...
    }
}
```

## Architecture & Concepts
The KnockbackSystems class is a static container acting as a namespace for two distinct but related Entity Component Systems (ECS). These systems implement the server-side logic for applying knockback forces to entities. The design separates the handling of knockback for standard entities (NPCs, monsters) from players, allowing for specialized logic such as server-side prediction for a smoother player experience.

The core architectural pattern is a data-driven ECS approach:
1.  **Data Component:** A KnockbackComponent is added to an entity, containing the magnitude, duration, and type of knockback to be applied. This component acts as a temporary "request" for a physics change.
2.  **Logic System:** The systems within KnockbackSystems query for entities that possess this KnockbackComponent.
3.  **State Mutation:** The system's tick method reads the data from the KnockbackComponent and translates it into a permanent change on another component, typically the Velocity component. It then manages the lifecycle of the KnockbackComponent, removing it after its effect is applied.

This separation of concerns is critical. Other game logic systems, such as the damage or combat systems, do not need to know how to manipulate physics directly. They only need to create and attach a KnockbackComponent to an entity to trigger the desired effect.

### ApplyKnockback System
This system provides the baseline knockback implementation for all non-player living entities. Its logic is direct and deterministic: it reads the knockback data and adds a velocity instruction to the entity's Velocity component.

### ApplyPlayerKnockback System
This system is a specialized implementation exclusively for player entities. It introduces a significant architectural feature: a toggleable server-side prediction model controlled by the static boolean **DO_SERVER_PREDICTION**.

-   **Prediction Disabled:** When false, the system behaves similarly to the non-player version, directly modifying the player's Velocity component. This is simpler but can lead to network artifacts where the client and server disagree on the player's position post-knockback.
-   **Prediction Enabled:** When true, the system does not modify the Velocity component. Instead, it writes the intended knockback to a dedicated KnockbackSimulation component. This decouples the knockback *intent* from its *application*. A separate, more complex physics reconciliation system is expected to consume the KnockbackSimulation component, blending the server's authoritative knockback with the client's continuous input to produce a smoother, more accurate result for the player.

The implementation of the IVelocityModifyingSystem interface signals to the engine's scheduler that these systems are participants in the physics pipeline and must be executed in a specific order relative to other physics calculations.

## Lifecycle & Ownership
-   **Creation:** Instances of the inner system classes (ApplyKnockback, ApplyPlayerKnockback) are created and managed by the server's central ECS engine during its initialization phase. They are registered within the engine's system graph.
-   **Scope:** A single instance of each system persists for the entire lifetime of the server process. The systems themselves are stateless; all state is stored in the components they operate on.
-   **Destruction:** The systems are destroyed when the server shuts down and the ECS engine is dismantled.

## Internal State & Concurrency
-   **State:** The systems are fundamentally stateless. They do not store or cache any data between ticks. All operations are performed on the components passed into the tick method for a given entity.
-   **Thread Safety:** The systems are designed for concurrent execution. The ApplyPlayerKnockback system explicitly implements the isParallel method, allowing the ECS engine to process different chunks of player entities across multiple threads simultaneously. This is safe because each invocation of the tick method operates on a unique entity and its components, preventing data races. The CommandBuffer provided to the tick method is a thread-safe mechanism for queueing component changes to be applied later in a synchronized step.

## API Surface
The public contract for these systems is defined by the EntityTickingSystem interface, which is invoked by the ECS engine.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getQuery() | Query | O(1) | Returns the component query that determines which entities this system will operate on. This is the primary filtering mechanism. |
| tick(...) | void | O(1) | Executes one unit of logic for a single entity per game tick. This method contains the core knockback application logic. |
| getGroup() | SystemGroup | O(1) | **(ApplyPlayerKnockback only)** Assigns the system to a specific execution group (InspectDamageGroup), ensuring it runs after damage is calculated but before physics integration. |

## Integration Patterns

### Standard Usage
The systems are not invoked directly. Instead, developers trigger them by adding a KnockbackComponent to an entity. The ECS engine automatically ensures the correct system's tick method is called for that entity on the subsequent server tick.

```java
// An external system (e.g., a damage system) adds a component to an entity.
// This is the *only* supported way to trigger the knockback logic.

Entity entity = ...;
CommandBuffer commands = ...;

KnockbackComponent knockback = new KnockbackComponent();
knockback.setVelocity(new Vector3f(0, 5, 10));
knockback.setDuration(0.25f);

commands.addComponent(entity.getRef(), KnockbackComponent.getComponentType(), knockback);

// On the next tick, either ApplyKnockback or ApplyPlayerKnockback will automatically run.
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Do not use `new ApplyKnockback()`. The systems are managed entirely by the ECS engine. Manual instantiation will result in a non-functional object that is not registered to receive ticks.
-   **Manual Tick Invocation:** Never call the `tick` method directly. Doing so bypasses the ECS scheduler, query matching, and thread management, leading to unpredictable behavior and state corruption.
-   **Modifying Velocity Directly:** For knockback effects, other systems should not modify the Velocity component directly. They should always add a KnockbackComponent to ensure the effect is processed correctly, respecting player prediction logic and system ordering.

## Data Pipeline
The flow of data for a knockback event is orchestrated by the ECS framework.

> Flow:
> External Event (e.g., Explosion, Melee Hit) -> Damage System -> **Adds KnockbackComponent to Entity** -> ECS Scheduler matches Entity to Query -> **KnockbackSystems.tick()** -> Reads KnockbackComponent -> **Writes to Velocity or KnockbackSimulation Component** -> Physics Engine integrates new velocity -> **KnockbackSystems.tick() removes KnockbackComponent via CommandBuffer**

