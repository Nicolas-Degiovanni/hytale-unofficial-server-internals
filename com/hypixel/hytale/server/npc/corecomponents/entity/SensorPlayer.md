---
description: Architectural reference for SensorPlayer
---

# SensorPlayer

**Package:** com.hypixel.hytale.server.npc.corecomponents.entity
**Type:** Component

## Definition
```java
// Signature
public class SensorPlayer extends SensorEntityBase {
```

## Architecture & Concepts
The SensorPlayer class is a specialized component within the server-side NPC AI framework. It serves as a concrete implementation of the abstract SensorEntityBase, designed exclusively for the perception of player entities.

Architecturally, this class embodies the **Strategy Pattern**. The generic entity sensing and prioritization logic resides within the parent SensorEntityBase. SensorPlayer provides a specific filtering strategy by overriding key methods, instructing the base sensor to focus its detection capabilities solely on players while ignoring all other entity types, including other NPCs.

This component is fundamental to any NPC behavior that requires awareness of nearby players, such as aggro mechanics, companion following, or scripted event triggers. It acts as the NPC's "eyes" for players, feeding a filtered list of perceived entities into higher-level AI decision-making systems like Behavior Trees or Goal-Oriented Action Planners.

### Lifecycle & Ownership
- **Creation:** SensorPlayer instances are not created directly. They are instantiated by the NPC asset loading pipeline via a corresponding `BuilderSensorPlayer`. This process occurs when an NPC entity is being assembled from its definition file, and that definition specifies a player-sensing component.
- **Scope:** The lifecycle of a SensorPlayer is strictly bound to the lifecycle of the NPC entity it is attached to. It persists as long as the parent NPC exists in the game world.
- **Destruction:** The component is marked for garbage collection when its parent NPC entity is despawned or destroyed. No manual cleanup is required.

## Internal State & Concurrency
- **State:** This class is **stateless and immutable**. Its purpose is to provide a fixed configuration to its parent. All mutable state, such as the list of currently detected players, is managed and owned by the parent SensorEntityBase.
- **Thread Safety:** This component is **not thread-safe** and is designed to be accessed exclusively from the main server game thread (tick loop). The parent NPC entity guarantees that all its components are updated sequentially within a single thread. Unsynchronized access from other threads will lead to world state corruption and server instability.

## API Surface
The public contract of SensorPlayer is primarily for internal framework use. The key methods are overrides that fulfill the contract required by SensorEntityBase.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| isGetPlayers() | boolean | O(1) | **Framework Internal.** Returns true, signaling the base sensor to include player entities in its perception checks. |
| isGetNPCs() | boolean | O(1) | **Framework Internal.** Returns false, signaling the base sensor to ignore other NPCs. |

## Integration Patterns

### Standard Usage
Developers do not interact with this class directly in code. Instead, its usage is declared within an NPC's asset definition file (e.g., a JSON or HOCON file). The game's asset loader and NPC factory are responsible for correct instantiation.

An AI behavior, such as a Behavior Tree node, would then query the parent sensor component for results.

```java
// Example of a higher-level AI system using the sensor's data
// Note: The code interacts with the base type, not SensorPlayer directly.

SensorEntityBase sensor = npc.getComponent(SensorEntityBase.class);
Optional<Entity> nearestPlayer = sensor.getNearestEntity();

if (nearestPlayer.isPresent()) {
    // Initiate combat or other player-aware behavior
    npc.getBrain().setTarget(nearestPlayer.get());
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never call `new SensorPlayer()`. The component requires a complex builder chain for proper initialization and integration into the NPC's component hierarchy. Bypassing the framework's factories will result in a non-functional, detached component.
- **Runtime Modification:** Do not attempt to use reflection or other means to change the return values of `isGetPlayers` or `isGetNPCs` after instantiation. The behavior of this component is intended to be fixed for its entire lifetime. For dynamic sensing behavior, use a different, more flexible sensor component.

## Data Pipeline
SensorPlayer acts as a filter within the NPC's perception data pipeline. It does not generate data itself but rather configures the process that does.

> Flow:
> World Spatial Query (e.g., Octree) -> Raw Nearby Entity List -> **SensorPlayer** (Filter Configuration) -> SensorEntityBase (Filtering Logic) -> Prioritized Player List -> AI Brain (Behavior Tree)

