---
description: Architectural reference for CachedPositionProvider
---

# CachedPositionProvider

**Package:** com.hypixel.hytale.server.npc.sensorinfo
**Type:** Transient

## Definition
```java
// Signature
public class CachedPositionProvider extends PositionProvider {
```

## Architecture & Concepts
The CachedPositionProvider is a specialized implementation of the PositionProvider contract. Its primary architectural role is not to store a position itself—that functionality is inherited—but to augment positional data with critical metadata: its origin.

This class acts as a state carrier within the NPC AI and sensor systems. It provides a standardized way for different parts of the AI to understand the reliability of a given location. By flagging a position as originating from a cache, downstream systems, such as behavior trees or pathfinders, can make more intelligent decisions. For example, an AI might initiate a "search" behavior if its target's position is from a cache, whereas it would engage in direct "pursuit" if the position is known to be live.

It is a fundamental component for creating believable AI that can handle uncertainty and stale information, preventing NPCs from having perfect, real-time knowledge of their environment.

### Lifecycle & Ownership
- **Creation:** Instantiated on-demand by NPC sensor or memory components. A common creator is a sensor that queries an entity's last known location from a world cache when the entity is not in direct line-of-sight.
- **Scope:** Ephemeral and short-lived. An instance typically exists only for the duration of a single AI processing tick. It is passed from the sensor, through the decision-making logic, and is discarded afterward.
- **Destruction:** Managed entirely by the Java garbage collector. Once the AI tick completes and all references are dropped, the object is eligible for cleanup.

## Internal State & Concurrency
- **State:** Mutable. The core state is a single boolean flag, `fromCache`, which is explicitly set after instantiation. The positional data it represents is inherited from the parent PositionProvider.
- **Thread Safety:** This class is **not thread-safe**. It contains no internal locking mechanisms. It is designed to be created, modified, and read within the confines of a single, dedicated server thread responsible for a specific NPC's AI updates. Unsynchronized access from multiple threads will result in race conditions and unpredictable AI behavior.

## API Surface
The public API is minimal, focusing exclusively on managing the cache status flag. Positional methods are inherited from the parent class.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| setIsFromCache(boolean) | void | O(1) | Sets the internal flag to indicate if the position data was sourced from a cache. |
| isFromCache() | boolean | O(1) | Returns true if the position data is stale and was retrieved from a cache. |

## Integration Patterns

### Standard Usage
This provider is used by AI systems to qualify the source of a target's location. The consumer of the provider must always check the `isFromCache` status before committing to an action.

```java
// An NPC's sensor system provides a position
PositionProvider targetLocation = findTargetPosition(targetEntity);

// Downstream behavior logic checks the data's reliability
if (targetLocation instanceof CachedPositionProvider) {
    if (((CachedPositionProvider) targetLocation).isFromCache()) {
        // Position is stale, initiate a search pattern
        npc.getBehaviorController().startSearchBehavior(targetLocation);
    } else {
        // Position is live, engage directly
        npc.getBehaviorController().startPursuitBehavior(targetLocation);
    }
}
```

### Anti-Patterns (Do NOT do this)
- **Ignoring the Flag:** The most critical anti-pattern is consuming the position data without checking the `isFromCache` flag. This will cause NPCs to treat stale, last-known locations as if they are real-time, leading to flawed behaviors like attacking an empty location where a player used to be.
- **State Re-use:** Do not hold references to a CachedPositionProvider instance across multiple AI ticks. Its state is only valid for the tick in which it was created. Reusing an instance can lead to its `fromCache` status being incorrect for a subsequent evaluation.

## Data Pipeline
The CachedPositionProvider serves as a data-enrichment step in the NPC's perception pipeline. It adds context to raw positional data before it is consumed by the decision-making layer.

> Flow:
> NPC Sensor Query -> World Entity Cache -> **CachedPositionProvider** (Instantiation & Flagging) -> NPC Behavior Tree -> Action Node (e.g., MoveTo, Search)

