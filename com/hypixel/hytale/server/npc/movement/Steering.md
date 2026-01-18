---
description: Architectural reference for Steering
---

# Steering

**Package:** com.hypixel.hytale.server.npc.movement
**Type:** Transient

## Definition
```java
// Signature
public class Steering {
```

## Architecture & Concepts
The Steering class is a fundamental data structure that encapsulates a movement and rotation *intent* for a server-side NPC. It is not a component or a service, but rather a transient data-transfer object that decouples high-level AI decision-making from low-level physics and entity actuation.

AI systems, such as Behavior Trees or Goal-Oriented Action Planners, do not directly modify an NPC's velocity or orientation. Instead, they compute a desired outcome and populate a Steering object to represent that intent. This object is then passed to a central movement processing system which interprets the intent, applies constraints (like collision and physics), and updates the NPC's final state for the current game tick.

A key design feature is the use of boolean flags like *hasTranslation* and *hasYaw*. This allows for the composition of behaviors. One behavior might only be concerned with facing a target (setting only yaw), while another handles locomotion (setting only translation). A movement aggregator can then combine these partial intents into a single, coherent action.

The static final field **Steering.NULL** represents the absence of any movement intent and is an implementation of the Null Object pattern. It provides a safe, non-null default that results in no action.

## Lifecycle & Ownership
- **Creation:** Steering objects are designed to be ephemeral. They are typically instantiated on the stack (`new Steering()`) at the beginning of an NPC's AI evaluation cycle for a single game tick. AI behaviors or pathfinding algorithms are the primary creators.

- **Scope:** The lifetime of a Steering object is extremely short, usually confined to a single method scope or, at most, the duration of one server tick for a specific NPC. They are not designed to be stored or referenced across ticks.

- **Destruction:** Once the NPC's movement system has consumed the Steering object and applied the resulting forces, the object is no longer referenced and becomes eligible for garbage collection.

**WARNING:** Due to their high-frequency creation, systems using this class are prime candidates for object pooling optimizations to reduce GC pressure. If a pool is used, ownership is transferred back to the pool after consumption.

## Internal State & Concurrency
- **State:** The Steering class is highly mutable. Its public API is designed as a fluent builder, where methods like setTranslation and setYaw modify internal state and return the instance for chaining. The state is not considered valid or complete until the AI logic has finished populating it.

- **Thread Safety:** This class is **not thread-safe** and provides no internal synchronization. It is designed to be confined to the single thread responsible for updating the NPC that owns it. Concurrent modification from multiple threads will result in race conditions and undefined behavior.

**WARNING:** Never share a Steering instance across threads. Do not read from a Steering object while another thread might be writing to it.

## API Surface
The public API is designed for building a movement command. Getters are provided but the primary interaction is through setters.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| clear() | Steering | O(1) | Resets all translation and rotation state, effectively returning the object to a default, inactive state. |
| assign(Steering other) | Steering | O(1) | Performs a deep copy of all fields from another Steering object into this one. |
| setTranslation(x, y, z) | Steering | O(1) | Sets the desired movement vector and marks translation as active. |
| setYaw(angle) | Steering | O(1) | Sets the desired absolute yaw and marks yaw as active. |
| hasTranslation() | boolean | O(1) | Returns true if a translation vector has been set since the last clear. |
| hasYaw() | boolean | O(1) | Returns true if a yaw value has been set since the last clear. |

## Integration Patterns

### Standard Usage
A standard use case involves an AI behavior calculating a movement vector and returning a populated Steering object to the core AI system for processing.

```java
// Inside an AI behavior class, e.g., FleeBehavior.evaluate()
public Steering evaluate(NpcContext context) {
    Vector3d fleeVector = calculateFleeVector(context);

    // Create a new, temporary steering command
    Steering intent = new Steering();

    // Populate the command with the desired movement
    if (fleeVector.isNonZero()) {
        intent.setTranslation(fleeVector);
        intent.setRelativeTurnSpeed(1.5); // Turn faster when fleeing
    }

    // The movement system will consume this returned object
    return intent;
}
```

### Anti-Patterns (Do NOT do this)
- **State Caching:** Do not store a Steering object as a member variable in a long-lived class and modify it over time. It is meant to be a fresh command for each tick.

- **Cross-Thread Modification:** Do not pass a Steering object to another thread for population or analysis. All operations must occur on the owning NPC's update thread.
    ```java
    // DANGEROUS: Modifying from a separate thread
    Steering sharedIntent = new Steering();
    executor.submit(() -> {
        // This will cause a race condition with the main game loop
        sharedIntent.setTranslation(1, 0, 0);
    });
    ```

- **Modifying the NULL object:** The static Steering.NULL instance is a shared sentinel. Modifying it will cause catastrophic, difficult-to-debug side effects for all other code that uses it.
    ```java
    // DO NOT DO THIS
    Steering.NULL.setYaw(90.0f); // This globally changes the NULL object
    ```

## Data Pipeline
The Steering object is a critical link in the NPC movement data pipeline, acting as the standardized message between intent and actuation.

> Flow:
> AI Behavior Logic -> **Steering (Movement Intent)** -> Movement Aggregator -> Physics & Collision System -> Final NPC Transform Update

