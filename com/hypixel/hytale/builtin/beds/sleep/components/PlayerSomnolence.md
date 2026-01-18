---
description: Architectural reference for PlayerSomnolence
---

# PlayerSomnolence

**Package:** com.hypixel.hytale.builtin.beds.sleep.components
**Type:** Transient

## Definition
```java
// Signature
public class PlayerSomnolence implements Component<EntityStore> {
```

## Architecture & Concepts
PlayerSomnolence is a data component within the Hytale Entity-Component-System (ECS) framework. It serves as a state container, exclusively designed to track an entity's sleep status. It does not contain any logic; its sole responsibility is to hold a reference to a PlayerSleep state object.

This component is a fundamental building block for the sleep gameplay mechanic. Systems, such as a hypothetical SleepSystem, query for entities possessing this component to enact sleep-related logic, such as advancing time, applying buffs, or changing player animations. Its existence on an entity signals that the entity is capable of sleeping.

The static method getComponentType is the primary integration point, providing the unique identifier used by the ECS framework to manage and query for this specific component type.

## Lifecycle & Ownership
- **Creation:** Instances are created and attached to an entity by a managing system, typically when sleep-related behavior is initiated (e.g., a player interacting with a bed). The static instance PlayerSomnolence.AWAKE serves as a convenient, shared default state for newly created player entities. The clone method facilitates entity duplication and state replication.
- **Scope:** The lifecycle of a PlayerSomnolence instance is strictly bound to the entity it is attached to. It persists as long as the parent entity exists within the EntityStore.
- **Destruction:** The component is marked for garbage collection and destroyed when its parent entity is removed from the world. There is no manual cleanup required.

## Internal State & Concurrency
- **State:** The component's state is mutable. It encapsulates a single reference, *state*, which points to a PlayerSleep object. While this class provides no public setters, the owning system is expected to replace the entire component on an entity to transition its state.
- **Thread Safety:** This component is **not thread-safe**. As with most ECS components, it is designed to be accessed and modified exclusively by its managing system on a single thread, typically the main server thread. Unsynchronized access from other threads will lead to race conditions and undefined behavior.

## API Surface
The public contract is minimal, reflecting its role as a simple data holder.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getComponentType() | static ComponentType | O(1) | Retrieves the registered type identifier for this component. |
| getSleepState() | PlayerSleep | O(1) | Returns the current sleep state of the entity. |
| clone() | Component | O(1) | Creates a shallow copy of the component, sharing the same state object reference. |

## Integration Patterns

### Standard Usage
Systems should retrieve this component from an entity to read its sleep state. To modify the state, a system should create a *new* PlayerSomnolence component and replace the existing one on the entity.

```java
// A system processing a player entity
PlayerSomnolence somnolence = entity.getComponent(PlayerSomnolence.getComponentType());

if (somnolence != null) {
    PlayerSleep currentState = somnolence.getSleepState();
    // ... read state and perform logic

    // To change state, create and attach a new component
    if (shouldStartSleeping) {
        PlayerSomnolence sleepingComponent = new PlayerSomnolence(PlayerSleep.Sleeping.INSTANCE);
        entity.setComponent(PlayerSomnolence.getComponentType(), sleepingComponent);
    }
}
```

### Anti-Patterns (Do NOT do this)
- **State Modification via Reflection:** Do not use reflection to modify the internal *state* field directly. This breaks the intended data flow of the ECS and can lead to unpredictable state transitions.
- **Modifying Static AWAKE Instance:** The static PlayerSomnolence.AWAKE instance is a shared default. Any modification to its internal state will affect all parts of the engine that use it as a default, causing catastrophic and difficult-to-diagnose bugs. Treat it as a read-only constant.
- **Cross-Thread Access:** Never read or attempt to replace this component from a thread other than the one designated for its managing system. All interactions must be synchronized through the main game loop.

## Data Pipeline
PlayerSomnolence acts as a state repository in the data flow, not a pipeline processor. It is written to by systems reacting to input and read by systems that produce output.

> Flow:
> Player Input (e.g., Use Bed) -> SleepSystem -> **PlayerSomnolence (State Updated)** -> AnimationSystem / TimeSystem (State Read) -> Visual/World Change

