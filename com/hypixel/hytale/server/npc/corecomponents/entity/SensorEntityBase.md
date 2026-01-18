---
description: Architectural reference for SensorEntityBase
---

# SensorEntityBase

**Package:** com.hypixel.hytale.server.npc.corecomponents.entity
**Type:** Framework Component (Abstract Base Class)

## Definition
```java
// Signature
public abstract class SensorEntityBase extends SensorWithEntityFilters {
```

## Architecture & Concepts
SensorEntityBase is an abstract foundational component within the NPC AI perception system. It does not represent a complete sensor but provides the core framework and reusable logic for any sensor designed to detect other entities, such as players or other NPCs.

Its primary architectural role is to orchestrate a multi-stage entity detection and filtering pipeline. It acts as a template that delegates specific logic to subclasses and injected strategy objects. This class embodies the **Template Method** design pattern, where the main detection algorithm is defined in the **matches** method, but specific steps like what entities to query (**isGetPlayers**, **isGetNPCs**) are left to concrete implementations.

It collaborates with several key systems:
- **Role:** The central component managing an NPC's AI state, behaviors, and sensors. SensorEntityBase is owned and driven by a Role.
- **PositionCache:** An optimized spatial query service provided by the Role. SensorEntityBase leverages this to efficiently find entities within a given range, avoiding a full world scan.
- **ISensorEntityPrioritiser:** A strategy interface for ranking potential targets. This decouples the sensor's detection logic from the criteria used to select the "best" target.
- **ISensorEntityCollector:** A strategy interface for gathering entities that pass initial filters, allowing for complex selection logic beyond just the highest priority target.
- **EntityStore:** The underlying Entity Component System (ECS) storage, which the sensor queries for component data like **TransformComponent** or **DeathComponent**.

## Lifecycle & Ownership
- **Creation:** Instances of SensorEntityBase subclasses are not created directly via code. They are instantiated by the NPC asset loading pipeline, specifically through a corresponding **BuilderSensorEntityBase**. This builder reads configuration from asset files (e.g., JSON) and injects dependencies like the **ISensorEntityPrioritiser**.
- **Scope:** The sensor's lifecycle is tightly bound to its owning **Role** component. It persists as long as the NPC entity exists and its Role is active.
- **Destruction:** The sensor is eligible for garbage collection when the NPC's **Role** is unloaded or the NPC entity is removed from the world. The **unloaded** and **removed** lifecycle methods are called to allow for state cleanup in its dependent strategies.

## Internal State & Concurrency
- **State:** Highly mutable. It maintains configuration state loaded from assets (e.g., **range**, **lockOnTarget**, **minRange**) and runtime state. Its most important mutable state is the **EntityPositionProvider**, which holds the result of the last sensor evaluation.
- **Thread Safety:** This class is **not thread-safe** and must only be accessed from the main server thread that ticks the NPC logic. Its methods operate on a **Store** object, which represents a state snapshot for the current tick. Concurrently calling its methods, especially **matches**, will lead to race conditions and unpredictable AI behavior. All operations are assumed to be synchronous and single-threaded.

## API Surface
The public contract is primarily defined by the **Sensor** interface and its lifecycle methods.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| matches(ref, role, dt, store) | boolean | O(N) | The core evaluation method. Returns true if a valid target is found. Complexity is proportional to the number of entities (N) within the sensor's range in the PositionCache. |
| getSensorInfo() | InfoProvider | O(1) | Returns the **EntityPositionProvider**, which contains the reference to the detected target entity. |
| registerWithSupport(role) | void | O(1) | Lifecycle callback. Pre-caches requirements with the Role's **PositionCache** to optimize spatial queries. |
| loaded(role) | void | O(1) | Lifecycle callback. Propagates the loaded event to internal strategies. |
| spawned(role) | void | O(1) | Lifecycle callback. Propagates the spawned event to internal strategies. |
| unloaded(role) | void | O(1) | Lifecycle callback. Propagates the unloaded event to internal strategies for cleanup. |
| done() | void | O(1) | Called after a sensor evaluation cycle to clear internal state, primarily the **positionProvider**. |

## Integration Patterns

### Standard Usage
A developer does not use this class directly. The standard pattern is to extend it to create a concrete sensor that specifies which entity types to target.

```java
// Example of a concrete implementation
public class SensorPlayers extends SensorEntityBase {
    // Constructor passes builder and dependencies to super
    public SensorPlayers(...) {
        super(...);
    }

    @Override
    protected boolean isGetPlayers() {
        return true; // This sensor specifically targets players
    }

    @Override
    protected boolean isGetNPCs() {
        return false; // And ignores other NPCs
    }
}
```
This new class would then be configured in an NPC asset file and instantiated by the engine.

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never call `new SensorEntityBaseSubclass()`. The object is complex and requires dependencies to be injected by the asset loading framework. Manual creation will result in a non-functional sensor.
- **State Caching:** Do not cache the result of **getSensorInfo** across ticks. The detected target is only valid for the duration of the tick in which **matches** returned true. The state is cleared via the **done** method.
- **Runtime Configuration Changes:** Modifying public fields like **range** or **lockOnTarget** after initialization is unsupported. These are considered immutable configuration properties and changing them at runtime can lead to inconsistent state with the **PositionCache**.

## Data Pipeline
The flow of data during a single sensor tick is a critical filtering process.

> Flow:
> NPC Role Tick -> **SensorEntityBase.matches()** -> Role.PositionCache Query -> Raw Entity List -> **Range & Filter Checks** -> ISensorEntityCollector -> ISensorEntityPrioritiser -> Final Target Ref -> **EntityPositionProvider** -> AI Behavior System

