---
description: Architectural reference for SensorEntity
---

# SensorEntity

**Package:** com.hypixel.hytale.server.npc.corecomponents.entity
**Type:** Transient Component

## Definition
```java
// Signature
public class SensorEntity extends SensorEntityBase {
```

## Architecture & Concepts
The SensorEntity is a fundamental component within the server-side NPC Artificial Intelligence framework. It acts as a specialized data provider, responsible for detecting and filtering entities within an NPC's environment. This class does not perform the actual spatial queries; rather, it holds the configuration that governs *what* the underlying sensing system should look for.

In the context of an NPC's "brain", the SensorEntity is a low-level perception module. Higher-level AI constructs, such as Behavior Trees or Goal Planners, consume the data produced by this sensor to make tactical decisions. For example, a hostile NPC might use a SensorEntity configured to find players to decide when to initiate combat, while a passive animal might use one to find other members of its species to flock with.

This class represents a specific, configured instance of a sensor, derived from a more generic SensorEntityBase. Its primary role is to apply boolean filters to the entity detection process.

## Lifecycle & Ownership
- **Creation:** A SensorEntity is instantiated by the NPC asset loading pipeline. The constructor's reliance on a BuilderSensorEntity and BuilderSupport indicates that its configuration is defined declaratively in an asset file (e.g., a JSON definition for a specific NPC type). It is never created manually during gameplay.
- **Scope:** The lifecycle of a SensorEntity is tightly bound to the lifecycle of the NPC entity it is attached to. It is created when the NPC is spawned and persists for the NPC's entire existence.
- **Destruction:** The object is eligible for garbage collection when its parent NPC entity is despawned or destroyed. There are no explicit cleanup or disposal methods.

## Internal State & Concurrency
- **State:** The state of the SensorEntity class itself is immutable. The configuration fields getPlayers, getNPCs, and excludeOwnType are final and set only once during construction. However, the parent class SensorEntityBase may contain mutable state, such as a cached list of detected entities that is updated on each AI tick.
- **Thread Safety:** This class is **not thread-safe**. All interactions with NPC components, including sensors, must be performed on the main server thread. The internal state of the parent class and the associated prioritiser is not designed for concurrent access. Modifying or accessing this component from worker threads will lead to race conditions and world state corruption.

## API Surface
The public API of this specific subclass is limited to accessing its configuration. The core sensing logic is inherited from SensorEntityBase.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| isGetPlayers() | boolean | O(1) | Returns true if this sensor is configured to detect player entities. |
| isGetNPCs() | boolean | O(1) | Returns true if this sensor is configured to detect non-player character entities. |
| isExcludingOwnType() | boolean | O(1) | Returns true if this sensor should filter out other NPCs of the same type as its owner. |

## Integration Patterns

### Standard Usage
This component is not meant to be used directly. It is automatically managed and invoked by a parent AI controller or "brain" component. The brain component calls a generic update or sense method (defined in SensorEntityBase) which in turn uses the configuration from this class to perform its work.

A conceptual example of how the AI system would use it:
```java
// Inside an AI Brain component's update loop
// This is a conceptual example; direct access is discouraged.

List<Entity> nearbyEntities = world.getEntitiesNear(this.npc);
List<Entity> filteredEntities = new ArrayList<>();

SensorEntity sensor = this.npc.getComponent(SensorEntity.class);

for (Entity entity : nearbyEntities) {
    if (entity.isPlayer() && sensor.isGetPlayers()) {
        filteredEntities.add(entity);
    }
    if (entity.isNPC() && sensor.isGetNPCs()) {
        if (sensor.isExcludingOwnType() && entity.getType() == this.npc.getType()) {
            continue; // Skip same-type NPC
        }
        filteredEntities.add(entity);
    }
}

// The prioritiser from SensorEntityBase would then sort filteredEntities.
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never create an instance using `new SensorEntity()`. The required Builder objects are part of a complex asset loading system. Bypassing this system will result in a misconfigured and non-functional component.
- **External State Modification:** Do not attempt to alter the filter flags after construction via reflection. The sensor's configuration is intended to be static and defined by the NPC's asset files. Runtime modification can lead to unpredictable AI behavior.

## Data Pipeline
The SensorEntity acts as a filter within the AI's perception data pipeline. It refines a raw list of nearby entities into a contextually relevant list for decision-making.

> Flow:
> World Spatial Query -> Raw Entity List -> **SensorEntity (Filtering Logic)** -> Prioritiser (Sorting Logic) -> AI Behavior System

