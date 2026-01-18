---
description: Architectural reference for StepComponent
---

# StepComponent

**Package:** com.hypixel.hytale.server.npc.components
**Type:** Transient

## Definition
```java
// Signature
public class StepComponent implements Component<EntityStore> {
```

## Architecture & Concepts
The StepComponent is a data-only component within the server-side Entity-Component-System (ECS) architecture. It does not contain any logic. Its sole purpose is to attach a time interval, the *tick length*, to an entity, typically a Non-Player Character (NPC).

This component serves as a configuration parameter for systems that execute logic in discrete, periodic steps. For example, a PathfindingSystem or a BehaviorTreeSystem might query for entities with a StepComponent to determine how frequently to update their state or re-evaluate their goals. By decoupling the time interval from the system itself, different entity types can operate on different update frequencies using the same underlying logic systems.

It is a fundamental building block for creating time-based or rhythmic behaviors without coupling them to the main server game loop's tick rate.

### Lifecycle & Ownership
- **Creation:** Instantiated on-demand by entity factories or behavior initializers when an entity is created or assigned a new stepped behavior. It is not a globally managed service.
- **Scope:** The lifecycle of a StepComponent instance is strictly bound to the entity to which it is attached. It exists only as long as its parent entity is present in the world's EntityStore.
- **Destruction:** The component is marked for garbage collection implicitly when its parent entity is destroyed or when the component is removed from the entity. There is no manual destruction method.

## Internal State & Concurrency
- **State:** The StepComponent is **immutable**. Its internal state, the `tickLength`, is a final field set only during construction. The `clone` method returns a new instance rather than modifying the existing one.
- **Thread Safety:** This component is inherently **thread-safe**. Its immutability guarantees that multiple systems can read its `tickLength` value from different threads without risk of race conditions or data corruption. No synchronization primitives are required.

## API Surface
The public contract is minimal, focusing entirely on data provision and type identification for the ECS framework.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getComponentType() | static ComponentType | O(1) | Retrieves the unique type identifier from the NPCPlugin registry. Essential for ECS queries. |
| getTickLength() | float | O(1) | Returns the configured time interval for the step. |
| clone() | Component | O(1) | Creates a new StepComponent instance with identical state. Fulfills the Component interface contract. |

## Integration Patterns

### Standard Usage
A system responsible for periodic updates will query the EntityStore for an entity's StepComponent and use its value to drive its internal timer or logic.

```java
// Example within a hypothetical BehaviorSystem update loop
void processEntity(Entity entity) {
    StepComponent step = entity.getComponent(StepComponent.getComponentType());
    if (step != null) {
        float interval = step.getTickLength();
        // Use 'interval' to determine if enough time has passed
        // to execute the next behavior step for this entity.
    }
}
```

### Anti-Patterns (Do NOT do this)
- **State Modification:** Do not attempt to modify a StepComponent after creation. If an entity's step interval needs to change, the correct pattern is to remove the existing component and add a new one with the desired value.
- **Global Tick Rate Assumption:** Do not assume `tickLength` is related to the server's global tick rate. This value is an independent floating-point duration, often measured in seconds, specific to the entity's behavior.

## Data Pipeline
The StepComponent acts as a source of static configuration data for other systems. It does not process data itself.

> Flow:
> Entity Definition (e.g., JSON) -> Entity Factory -> **StepComponent** (instantiated and attached) -> EntityStore -> Behavior System (reads `tickLength`) -> Entity State Update (e.g., new position)

