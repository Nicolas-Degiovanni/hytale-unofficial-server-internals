---
description: Architectural reference for SensorBase
---

# SensorBase

**Package:** com.hypixel.hytale.server.npc.corecomponents
**Type:** Transient Component Base

## Definition
```java
// Signature
public abstract class SensorBase extends AnnotatedComponentBase implements Sensor {
```

## Architecture & Concepts
SensorBase is an abstract foundational class within the server-side Non-Player Character (NPC) artificial intelligence framework. It is not a component that is used directly, but rather serves as the parent for all concrete sensor implementations, such as a PlayerDetectionSensor or a LowHealthSensor.

Its primary architectural role is to provide a standardized contract and shared state management for the "Sense" phase of the classic "Sense-Think-Act" AI loop. A Sensor's purpose is to evaluate a specific condition within the game world state. The SensorBase class provides the core logic for a critical and common sensor behavior: the ability to trigger only once.

This component is a fundamental building block for constructing complex NPC behaviors. High-level AI constructs, such as Behavior Trees or Finite State Machines, query a collection of these sensors each tick to determine which conditions are met, thereby deciding which branch of logic or state transition to execute.

## Lifecycle & Ownership
- **Creation:** Instances of SensorBase subclasses are not created directly. They are instantiated by their corresponding builder, which extends BuilderSensorBase. This construction typically occurs during the server's asset loading phase, where NPC behavior profiles are parsed from configuration files and assembled into executable AI logic trees. The resulting Sensor object is owned by a higher-level AI component, such as a Role or a specific node in a behavior tree.
- **Scope:** The lifetime of a SensorBase instance is tightly coupled to the NPC entity that owns its behavior definition. It persists as long as the NPC is active in the world and its current AI behavior requires this specific sensor.
- **Destruction:** The object is eligible for garbage collection when the owning NPC is despawned or when its AI logic is reconfigured in a way that no longer includes this sensor. There is no explicit destruction or cleanup method.

## Internal State & Concurrency
- **State:** SensorBase maintains a mutable internal state. The final boolean field *once* is set at construction and determines if the sensor can be re-triggered. The boolean field *triggered* tracks the current activation state and is modified at runtime. This statefulness is essential for creating behaviors that react to an event only the first time it occurs.
- **Thread Safety:** This class is **not thread-safe**. All interactions with a SensorBase instance and its subclasses must be confined to the server's main entity update thread. Unsynchronized access from other threads will lead to race conditions, resulting in unpredictable and erroneous NPC behavior. The AI system design assumes single-threaded access per NPC.

## API Surface
The public API provides the essential contract for the AI engine to evaluate world conditions and manage trigger state.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| matches(ref, role, dt, store) | boolean | O(1) | Core evaluation method. Returns true if the sensor's conditions are met. This base implementation only checks the *once* flag; subclasses override this to inspect the world state. |
| clearOnce() | void | O(1) | Resets the triggered state, allowing a *once* sensor to fire again. This is typically managed by the AI instruction processor. |
| setOnce() | void | O(1) | Forcibly marks the sensor as triggered. External use is discouraged. |
| isTriggered() | boolean | O(1) | Returns the current value of the internal *triggered* flag. |
| processDelay(dt) | boolean | O(1) | A default implementation for delay processing. Subclasses may override to introduce time-based logic. Returns true by default. |

## Integration Patterns

### Standard Usage
A developer never uses SensorBase directly. Instead, they implement a concrete sensor and its corresponding builder. The AI engine then uses this implementation within a larger behavior definition.

```java
// 1. A concrete implementation (Hypothetical)
public class PlayerNearbySensor extends SensorBase {
    private final double radius;

    // Constructor uses a builder
    public PlayerNearbySensor(BuilderPlayerNearbySensor builder) {
        super(builder);
        this.radius = builder.getRadius();
    }

    @Override
    public boolean matches(Ref<EntityStore> ref, Role role, double dt, Store<EntityStore> store) {
        // First, check the base class logic for the "once" flag
        if (!super.matches(ref, role, dt, store)) {
            return false;
        }
        
        // Next, perform the actual world check
        // ... logic to find players within this.radius ...
        boolean playerIsNearby = world.findPlayerInRange(ref.get().getPosition(), this.radius);
        return playerIsNearby;
    }
}

// 2. The AI engine evaluates the sensor each tick
// This code would exist deep within the NPC behavior processing loop.
boolean conditionsMet = myPlayerNearbySensor.matches(npcRef, currentRole, deltaTime, worldStore);
if (conditionsMet) {
    // Mark as triggered and execute the associated action
    myPlayerNearbySensor.setOnce(); 
    executeCombatBehavior();
}
```

### Anti-Patterns (Do NOT do this)
- **State Tampering:** Do not manually call `clearOnce()` or `setOnce()` from peripheral game systems. These methods are intended for the exclusive use of the core AI instruction processor that manages the NPC's behavior lifecycle. External manipulation will break the logic of complex, multi-stage behaviors.
- **Ignoring Superclass Behavior:** When overriding the `matches` method in a subclass, failing to call `super.matches(...)` as the first step will bypass the essential "trigger once" logic, causing behaviors intended to run once to run every tick.

## Data Pipeline
SensorBase acts as a stateful filter in the data flow from the world state to the NPC action system.

> Flow:
> Raw World State (Entity Positions, Health, etc.) -> Concrete Sensor Implementation (e.g., PlayerNearbySensor) -> **SensorBase.matches()** -> Boolean Result -> AI Behavior Tree/FSM -> NPC Action (e.g., Move, Attack)

