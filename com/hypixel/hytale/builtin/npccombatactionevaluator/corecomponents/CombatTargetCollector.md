---
description: Architectural reference for CombatTargetCollector
---

# CombatTargetCollector

**Package:** com.hypixel.hytale.builtin.npccombatactionevaluator.corecomponents
**Type:** Transient

## Definition
```java
// Signature
public class CombatTargetCollector implements ISensorEntityCollector {
```

## Architecture & Concepts
The CombatTargetCollector is a specialized worker class within the NPC Artificial Intelligence framework, specifically serving the sensory and perception systems. It is not a standalone service but rather a pluggable implementation of the ISensorEntityCollector interface.

Its primary function is to act as a post-processor for a broader entity detection mechanism, such as a proximity sensor. When a sensor identifies a nearby entity, it delegates the classification and memory storage of that entity to a collector like this one. The CombatTargetCollector determines the relationship between the NPC and the detected entity by querying the world's **Attitude** system. Based on the result—HOSTILE, FRIENDLY, etc.—it populates the NPC's **TargetMemory** component. This component then serves as the short-term situational awareness database for higher-level AI decision-making systems, such as action selection and pathfinding.

This class forms a critical bridge between raw sensory input (lists of nearby entities) and contextual understanding (lists of known hostiles and friendlies).

## Lifecycle & Ownership
The lifecycle of a CombatTargetCollector instance is extremely short and tightly controlled by the parent NPC sensor system. It is a transient object, created and destroyed for a single perception update.

-   **Creation:** Instantiated on-demand by an NPC's sensor system at the beginning of a perception cycle. It is never pre-allocated or shared between NPCs or ticks.
-   **Scope:** The instance exists only for the duration of a single sensor evaluation pass. During this time, its internal state, such as `closestHostileDistanceSquared`, is accumulated.
-   **Destruction:** The `cleanup` method is invoked by the owning sensor system immediately after processing all potential targets for a given tick. This is a mandatory step to prevent state from one perception cycle from leaking into the next. The object is then eligible for garbage collection.

## Internal State & Concurrency
-   **State:** The CombatTargetCollector is highly stateful but its state is transient. It holds direct references to the NPC's `role` and `targetMemory` component, and maintains its own internal tracking state (`closestHostileDistanceSquared`) for the duration of a collection pass. This state is discarded upon cleanup.

-   **Thread Safety:** **This class is not thread-safe and must not be accessed concurrently.** It is designed to operate exclusively within the single-threaded context of an individual NPC's AI update loop. The owning sensor system is responsible for ensuring that all method calls on a given instance are serialized. Concurrent calls to `collectMatching` would introduce race conditions when updating `closestHostileDistanceSquared` and writing to the shared `TargetMemory` component.

## API Surface
The public API is defined by the ISensorEntityCollector interface and is exclusively intended for consumption by the NPC sensor framework.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| registerWithSupport(Role) | void | O(1) | Declares system-level dependencies. Requires the AttitudeCache to be available. |
| init(ref, role, accessor) | void | O(1) | Prepares the collector for a new pass by caching references to the NPC's Role and TargetMemory. |
| collectMatching(ref, targetRef, accessor) | void | O(1) | Core processing logic. Evaluates a single target entity and updates TargetMemory if applicable. |
| cleanup() | void | O(1) | Resets all internal state. **CRITICAL:** Must be called at the end of each perception cycle. |
| terminateOnFirstMatch() | boolean | O(1) | Always returns false (if initialized), ensuring the sensor processes all potential targets. |

## Integration Patterns

### Standard Usage
This class is not intended for direct use by game logic developers. It is configured declaratively as part of an NPC's behavior definition and managed entirely by the engine's sensor systems. The framework is responsible for the full lifecycle.

A conceptual example of how the *engine* would use it:
```java
// Engine-level code, NOT for general use
ISensorEntityCollector collector = new CombatTargetCollector();
collector.init(npcRef, npcRole, componentAccessor);

for (Ref<EntityStore> target : nearbyEntities) {
    collector.collectMatching(npcRef, target, componentAccessor);
}

collector.cleanup();
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Never call `new CombatTargetCollector()` in game logic. This bypasses the sensor framework and its lifecycle guarantees.
-   **State Leakage:** Failing to call `cleanup()` after a sensor pass is a critical error. This will cause stale data, such as the previous tick's closest hostile, to persist, leading to highly erratic and unpredictable AI behavior.
-   **Premature Access:** Calling `collectMatching` before `init` has been successfully invoked will result in a NullPointerException, as the essential `targetMemory` and `role` fields will be null.

## Data Pipeline
The CombatTargetCollector sits in the middle of the NPC perception data flow, transforming raw entity data into categorized memory.

> Flow:
> NPC Sensor System (e.g., Proximity) -> List of Entity References -> **CombatTargetCollector.collectMatching** -> Attitude System Query -> Update to NPC's TargetMemory Component -> AI Action Selection System

