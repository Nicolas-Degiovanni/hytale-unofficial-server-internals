---
description: Architectural reference for SensorFlockLeader
---

# SensorFlockLeader

**Package:** com.hypixel.hytale.server.flock.corecomponents
**Type:** Transient

## Definition
```java
// Signature
public class SensorFlockLeader extends SensorBase {
```

## Architecture & Concepts
The SensorFlockLeader is a specialized component within the server-side NPC Artificial Intelligence framework. Its primary function is to act as a sensory input provider that detects the leader of an entity's current flock. It is designed to be used as a conditional check within higher-level AI structures like Behavior Trees or Finite State Machines.

This sensor serves as a critical link between an entity's social group data (the FlockMembership and EntityGroup components) and its decision-making logic. When its `matches` method is evaluated, it queries the entity component system to find the flock, identify its leader, and store the leader's location. This result determines whether behaviors dependent on a leader, such as *Follow* or *Group Formation*, can be executed.

Unlike a simple boolean check, the SensorFlockLeader populates an internal InfoProvider with the leader's positional data. This allows subsequent AI components to consume the sensor's output directly, creating an efficient data flow where the act of sensing also prepares the necessary data for acting.

## Lifecycle & Ownership
- **Creation:** An instance of SensorFlockLeader is not created directly. It is instantiated via its corresponding builder, BuilderSensorFlockLeader, typically during the declarative setup of an NPC's AI profile or Role. This process usually occurs once when an NPC type is defined or loaded, not during the main game loop.
- **Scope:** The object's lifetime is bound to the AI configuration it is part of. It persists as long as the parent Role or behavior definition is attached to an NPC.
- **Destruction:** The instance is marked for garbage collection when the parent AI configuration is removed or replaced, for example, when an entity is despawned from the world.

## Internal State & Concurrency
- **State:** This class is stateful. It maintains a reference to an EntityPositionProvider, which caches the location of the flock leader. This state is highly volatile and is explicitly updated or cleared on every invocation of the `matches` method. A successful match populates the provider, while any failure in the logic chain—such as the entity not being in a flock or the flock having no leader—will clear it.

- **Thread Safety:** This component is **not thread-safe**. It is designed to be owned and operated exclusively by the server's main entity update thread. The internal state, positionProvider, is mutated without any synchronization mechanisms. Concurrent access would result in race conditions and unpredictable AI behavior.

    **Warning:** Never access or modify an instance of SensorFlockLeader from an asynchronous task or a different thread. All interactions must be synchronized with the main server tick.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| matches(ref, role, dt, store) | boolean | O(1) | Evaluates if the entity has a valid flock leader. Populates internal state on success or clears it on failure. This is the primary entry point for the AI system. |
| getSensorInfo() | InfoProvider | O(1) | Returns the internal position provider. This should only be called immediately after a successful `matches` call to retrieve the leader's location data. |

## Integration Patterns

### Standard Usage
The SensorFlockLeader is not intended to be called imperatively by general game logic. It is embedded within an AI definition and is ticked automatically by the AI scheduler. A behavior node would use it as a condition before attempting to execute a related action.

```java
// Conceptual example of how a Behavior Tree node might use the sensor.
// This code does not exist literally but illustrates the design pattern.

// 1. The sensor is evaluated as a condition.
boolean canSeeLeader = sensorFlockLeader.matches(entityRef, role, dt, store);

// 2. If the condition passes, the behavior proceeds.
if (canSeeLeader) {
    // 3. The data from the sensor is retrieved and used.
    EntityPositionProvider provider = (EntityPositionProvider) sensorFlockLeader.getSensorInfo();
    Vector3f leaderPosition = provider.getPosition();
    
    // Create a movement goal to follow the leader.
    movementGoal = new FollowEntityGoal(leaderPosition);
    return BehaviorStatus.SUCCESS;
} else {
    // The entity cannot see a leader, so this branch of the tree fails.
    return BehaviorStatus.FAILURE;
}
```

### Anti-Patterns (Do NOT do this)
- **State Caching:** Do not call `getSensorInfo` and cache the returned provider or its position across multiple server ticks. The leader's status or position can change at any moment. The `matches` method **must** be called on every tick where the leader's information is required to ensure data is current.
- **External Instantiation:** Do not use the BuilderSensorFlockLeader to create instances of this sensor outside of an AI Role or behavior definition file. The component's lifecycle is managed by the AI framework.
- **Ignoring Return Value:** Do not call `getSensorInfo` without first checking that `matches` returned true. If `matches` returns false, the data in the InfoProvider is invalid and has been explicitly cleared.

## Data Pipeline
The component functions as a processor in a data pipeline that transforms entity component data into actionable AI sensory information.

> Flow:
> Entity Component Store → `FlockMembership` Component → `EntityGroup` Component → **SensorFlockLeader** → `EntityPositionProvider` → AI Behavior Node (e.g., FollowGoal)

