---
description: Architectural reference for SensorDroppedItem
---

# SensorDroppedItem

**Package:** com.hypixel.hytale.server.npc.corecomponents.items
**Type:** Component

## Definition
```java
// Signature
public class SensorDroppedItem extends SensorBase {
```

## Architecture & Concepts

The SensorDroppedItem is a fundamental component within the Non-Player Character (NPC) sensory system. It provides NPCs with the ability to perceive and react to dropped item entities within the game world. This class encapsulates the logic for detecting items based on a configurable set of criteria, such as type, distance, and line of sight.

Architecturally, this sensor acts as a stateful predicate within an NPC's decision-making logic, typically a Behavior Tree or a Finite State Machine. It does not execute actions itself; rather, it answers the question: "Is there a noteworthy dropped item nearby according to my current rules?".

The component is heavily optimized to reduce server load. It integrates with the NPC's Role and its associated PositionCache. Instead of performing a costly broad-phase world query on every evaluation, it requests that the PositionCache maintain a pre-filtered list of nearby items, upon which this sensor performs its more detailed, fine-grained checks.

The result of a successful detection is stored internally in an EntityPositionProvider, which serves as a standardized data contract. Other AI components, such as Goals or Actions, can then consume this provider to get a stable reference to the detected item's location and entity, enabling behaviors like pathfinding to or picking up the item.

### Lifecycle & Ownership
- **Creation:** An instance of SensorDroppedItem is created by the server's asset loading pipeline when an NPC is initialized. Its configuration is defined in an NPC's asset files (e.g., JSON) and is parsed by a corresponding BuilderSensorDroppedItem, which then instantiates this class.
- **Scope:** The object's lifetime is tightly coupled to its parent NPC entity. It is instantiated when the NPC is spawned into the world and persists until the NPC is despawned or destroyed.
- **Destruction:** The object is eligible for garbage collection once its parent NPC entity is removed from the world. It does not manage any native resources and has no explicit destruction method.

## Internal State & Concurrency
- **State:** This class is stateful.
    - **Configuration State:** Fields such as *items*, *attitudes*, *range*, and *viewCone* are configured at creation and are treated as immutable during the sensor's lifetime.
    - **Runtime State:** The *heading* and *positionProvider* fields are mutable. They are updated on each invocation of the *matches* method to reflect the NPC's current orientation and the result of the latest detection query. The *positionProvider* caches a reference to the detected entity for the duration of a single game tick.

- **Thread Safety:** **This class is not thread-safe.** It is designed to be owned and operated exclusively by the server's main game loop thread that manages the parent NPC. All method calls must be synchronized with the world tick. Unsynchronized access from other threads will result in severe concurrency issues, including inconsistent state and world corruption.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| matches(ref, role, dt, store) | boolean | O(N) | Evaluates sensor conditions against the world. N is the number of items pre-filtered by the PositionCache. Returns true if a valid item is found. **Warning:** This method mutates internal state. |
| getSensorInfo() | InfoProvider | O(1) | Returns the data provider containing information about the item found during the last successful *matches* call. Returns stale or empty data if *matches* was not called or returned false in the current tick. |
| registerWithSupport(role) | void | O(1) | A one-time setup method called during NPC initialization. It primes the NPC's PositionCache to begin tracking dropped items within this sensor's range. |

## Integration Patterns

### Standard Usage

The SensorDroppedItem is designed to be used as a condition within a larger AI behavior. An action or goal should only consume the sensor's data after the sensor has been successfully evaluated in the same tick.

```java
// Conceptual example within an AI Behavior Tree node

// In a Condition Node:
boolean itemIsVisible = sensorDroppedItem.matches(npcRef, npcRole, deltaTime, worldStore);
if (itemIsVisible) {
    // Proceed to the Action Node
}

// In a subsequent Action Node (within the same tick):
InfoProvider provider = sensorDroppedItem.getSensorInfo();
// The provider is now guaranteed to hold a valid target
Vector3d itemLocation = provider.getPosition(worldStore);
// Create a goal to move the NPC to itemLocation
npc.getBrain().setGoal(new MoveToLocationGoal(itemLocation));
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never create an instance using `new SensorDroppedItem()`. The object is fundamentally dependent on its builder for proper configuration from asset files. A manually created instance will be in an invalid state and cause NullPointerExceptions.
- **State Caching Across Ticks:** Do not call `getSensorInfo` and cache the result for use in a future game tick. The validity of the contained entity reference is not guaranteed beyond the current tick. The `matches` method must be called every tick to refresh the sensor's state before its data is used.
- **Ignoring the `matches` Result:** Do not call `getSensorInfo` without first checking that `matches` returned true. If no item was detected, the provider will contain stale or null data, leading to unpredictable behavior.

## Data Pipeline

The flow of data begins with asset definition and culminates in the AI system making a decision.

> **Configuration Flow:**
> NPC Asset File (JSON) -> BuilderSensorDroppedItem -> **SensorDroppedItem Instance** (Immutable Configuration)

> **Runtime Evaluation Flow (per tick):**
> World State (All Dropped Items) -> PositionCache (Spatial Hash Query) -> `matches()` -> `filterItem()` (View Cone, LoS, Type Checks) -> **EntityPositionProvider (Mutable State)** -> `getSensorInfo()` -> AI Behavior Tree (Consumer)

