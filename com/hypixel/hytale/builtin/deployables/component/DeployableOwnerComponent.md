---
description: Architectural reference for DeployableOwnerComponent
---

# DeployableOwnerComponent

**Package:** com.hypixel.hytale.builtin.deployables.component
**Type:** Transient Component

## Definition
```java
// Signature
public class DeployableOwnerComponent implements Component<EntityStore> {
```

## Architecture & Concepts
The DeployableOwnerComponent is a data component within the Entity-Component-System (ECS) framework, attached to entities that can create and own other entities, known as "deployables" (e.g., turrets, mines, or temporary structures).

This component does not contain any logic for the act of deploying; rather, it serves as a server-side manifest for an entity's deployed assets. Its primary architectural role is to enforce gameplay rules by tracking and limiting the number of active deployables an owner can have in the world.

It manages two types of limits simultaneously:
1.  **Per-Type Limit:** Restricts the number of a specific deployable type (e.g., a player can only have two "Basic Turrets"). This is defined in the DeployableComponent of the deployable itself.
2.  **Global Limit:** Restricts the total number of deployables an owner can have active at once, regardless of type. This is sourced from the global GameplayConfig.

To maintain system stability and avoid race conditions, the component uses a deferred destruction pattern. When a limit is exceeded, the oldest deployable entity is not destroyed immediately. Instead, it is added to a destruction queue (`deployablesForDestruction`) which is processed during the component's next `tick` cycle.

## Lifecycle & Ownership
-   **Creation:** A DeployableOwnerComponent is not present on entities by default. It is instantiated and attached to an owner entity by a game system at the moment the owner successfully creates its first deployable entity.
-   **Scope:** The component's lifecycle is strictly tied to its owner entity. It persists as long as the owner entity exists in the world.
-   **Destruction:** The component is destroyed and its memory is reclaimed when its owner entity is removed from the world. Note that this component does not inherently manage the cleanup of its tracked deployables upon its own destruction; that is typically handled by higher-level world or entity management systems.

## Internal State & Concurrency
-   **State:** This component is highly mutable and stateful. Its internal lists tracking active and to-be-destroyed deployables are modified frequently during gameplay as deployables are created and removed.
-   **Thread Safety:** **This component is not thread-safe.** All operations must be performed on the main server game thread. The internal data structures (ObjectArrayList, Object2IntOpenHashMap) are not designed for concurrent access. Accessing this component from asynchronous tasks or other threads will lead to state corruption and server instability.

## API Surface
| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| tick(commandBuffer) | void | O(N) | Processes the destruction queue, where N is the number of entities marked for destruction. Issues death commands via the CommandBuffer. |
| registerDeployable(...) | void | O(M) | Registers a new deployable. Immediately runs limit checks and may queue older deployables for destruction. M is the current number of deployables owned. |
| deRegisterDeployable(...) | void | O(M) | De-registers a deployable, typically when it is destroyed by normal means. M is the current number of deployables owned. |
| clone() | Component | O(1) | **WARNING:** The current implementation is incorrect and returns a new KnockbackComponent, not a clone of itself. Do not use. |

## Integration Patterns

### Standard Usage
This component is managed by game systems that handle entity creation. A system responsible for placing a deployable would first create the entity, then retrieve the owner's DeployableOwnerComponent (or create it) and register the new deployable.

```java
// A system places a new deployable entity on behalf of an owner
Ref<EntityStore> ownerRef = ...;
Ref<EntityStore> newDeployableRef = ...;
DeployableComponent deployableComp = ...;
String deployableId = "basic_turret";

// Get or create the owner's tracking component
DeployableOwnerComponent ownerComp = store.getOrAddComponent(ownerRef, DeployableOwnerComponent.getComponentType());

// Register the new deployable, which triggers limit enforcement
ownerComp.registerDeployable(ownerRef, deployableComp, deployableId, newDeployableRef, store);
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Never use `new DeployableOwnerComponent()`. The component must be managed by the ECS `Store` to ensure it is correctly associated with an entity.
-   **Manual Destruction:** Do not manually remove entities from the internal `deployables` list. Always use the `deRegisterDeployable` method to ensure counts are updated correctly.
-   **Incorrect Cloning:** The `clone()` method is fundamentally broken and should not be called. Relying on it will lead to incorrect component data on cloned entities.
-   **Asynchronous Modification:** Do not call `registerDeployable` or `deRegisterDeployable` from outside the main game tick, as this will cause concurrency exceptions.

## Data Pipeline
The component's primary flow is a control loop for enforcing entity limits, not a traditional data transformation pipeline.

> Flow:
> Player Action -> Deployable Creation System -> **DeployableOwnerComponent.registerDeployable()** -> Limit Check Logic -> Oldest Deployable Ref added to `deployablesForDestruction` queue -> Game Loop Tick -> **DeployableOwnerComponent.tick()** -> CommandBuffer receives `DeathComponent` command -> Entity Death System -> Deployable Entity Removed from World

