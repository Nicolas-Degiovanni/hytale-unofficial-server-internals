---
description: Architectural reference for RoleSystems
---

# RoleSystems

**Package:** com.hypixel.hytale.server.npc.systems
**Type:** Utility / Namespace

## Definition
```java
// Signature
public class RoleSystems {
    // Contains multiple nested static System classes
}
```

## Architecture & Concepts

The RoleSystems class is not a traditional object but a static namespace that groups a collection of critical Entity Component Systems (ECS) responsible for managing the entire lifecycle and behavior of Non-Player Character (NPC) Roles. These systems form a sequential pipeline that executes during each server tick, ensuring that NPC logic is processed in a predictable and orderly fashion.

Each nested class within RoleSystems represents a distinct phase of the NPC update loop. They are discovered and managed by the server's main ECS scheduler and are not intended for direct invocation. The systems use ECS queries to efficiently operate on entities possessing an NPCEntity component, along with other required components.

The core architectural pattern is a dependency-ordered pipeline of systems:

1.  **Pre-Behaviour Tick:** Prepares the NPC for the main logic tick by validating targets and clearing transient state.
2.  **Behaviour Tick:** Executes the core, high-level decision-making logic contained within the NPC's assigned Role.
3.  **Post-Behaviour Tick:** Cleans up after the main logic tick, applies movement constraints, and resets per-tick data.

Additionally, separate systems handle the initial setup and final teardown of Roles when an NPC entity is created or destroyed, as well as optional debug rendering.

### Lifecycle & Ownership

-   **Creation:** The nested system classes within RoleSystems are instantiated by the server's ECS framework during the engine's bootstrap and system registration phase. They are not created manually.
-   **Scope:** Instances of these systems persist for the entire server session. They are effectively singletons managed by the ECS engine.
-   **Destruction:** The systems are destroyed when the server shuts down and the ECS world is torn down.

## Internal State & Concurrency

-   **State:** The RoleSystems class itself is stateless. Its nested system classes hold immutable references to ComponentType objects, which are configured upon creation. The primary state they operate on is the component data of the entities themselves, stored within the ECS Store.
-   **Thread Safety:** The systems are designed with concurrency in mind, but with varying strategies.
    -   **BehaviourTickSystem:** This system is **not parallelizable**. It aggregates all relevant entities into a single, ThreadLocal list before processing them sequentially. This design implies that the core Role logic may have side effects that are not safe for parallel execution.
    -   **Pre/Post/Debug Systems:** These systems extend SteppableTickingSystem and implement the isParallel method, allowing the ECS scheduler to process entity chunks across multiple worker threads for improved performance.
    -   **RoleActivateSystem:** As a HolderSystem, its onEntityAdd and onEntityRemoved methods are called by the ECS framework. Concurrency guarantees are provided by the framework, ensuring safe component access for the specific entity being processed.

## API Surface

The public API of RoleSystems is its collection of nested system classes, which are intended for registration with the ECS engine, not for direct calls.

| Symbol | Type | Role | Description |
| :--- | :--- | :--- | :--- |
| PreBehaviourSupportTickSystem | ECS System | Preparation | Runs before the main behavior tick. Validates entity targets and clears per-tick state like steering accumulators. |
| BehaviourTickSystem | ECS System | Core Logic | Executes the primary Role.tick method, which contains the NPC's AI and decision-making logic. |
| PostBehaviourSupportTickSystem | ECS System | Finalization | Runs after steering systems. Cleans up motion controller state, constrains rotations, and resets combat and world support data. |
| RoleActivateSystem | ECS System | Setup/Teardown | Reacts to the addition and removal of NPC entities. Activates and deactivates the Role and its associated controllers. |
| RoleDebugSystem | ECS System | Debugging | Invokes debug rendering logic for Roles that have an active RoleDebugDisplay. |

## Integration Patterns

### Standard Usage

A developer does not interact with RoleSystems directly. Instead, they create an entity and attach an NPCEntity component to it. The ECS framework automatically ensures that this entity is processed by the systems within RoleSystems on each server tick.

```java
// An entity is created elsewhere with an NPCEntity component.
// The RoleSystems.* systems will automatically find and process it.

// Example of entity creation (conceptual)
Entity newNpc = world.createEntity();
NPCEntity npcComponent = new NPCEntity(new MyCustomRole());
newNpc.addComponent(npcComponent);
newNpc.addComponent(new TransformComponent());
// ... other required components
```

### Anti-Patterns (Do NOT do this)

-   **Direct Instantiation:** Do not use `new RoleSystems.BehaviourTickSystem()`. The ECS framework is solely responsible for creating and managing system instances.
-   **Manual Invocation:** Never call the `tick` or `steppedTick` methods directly. The ECS scheduler invokes these methods at the correct time and on the correct thread, respecting all system dependencies. Violating this will break the execution order and lead to severe state corruption.
-   **Assuming Execution Order:** Do not write game logic that depends on an implicit execution order. The order is explicitly defined by the SystemDependency annotations. Rely on this declared contract, not on observed behavior.

## Data Pipeline

The systems within RoleSystems form a strict, ordered pipeline for processing NPC data each tick. The dependencies ensure that data flows logically from preparation, to execution, to finalization.

> Flow:
> **1. PreBehaviourSupportTickSystem** (Order.BEFORE BehaviourTickSystem)
>    - Cleans steering data.
>    - Validates targets, removing dead or invalid ones.
>    - Resets per-tick state on the NPCEntity component.
>
> **2. BehaviourTickSystem**
>    - Executes the main `Role.tick()` method.
>    - The Role populates steering requests (BodySteering, HeadSteering) and may nominate actions.
>
> **3. SteeringSystem** (External system, not in this class)
>    - Consumes steering requests and calculates velocity/rotation changes.
>
> **4. PostBehaviourSupportTickSystem** (Order.AFTER SteeringSystem)
>    - Clears motion controller overrides.
>    - Applies final rotation constraints to the TransformComponent.
>    - Ticks and updates various support modules (CombatSupport, WorldSupport).
>    - Resets transient data for the next frame.

