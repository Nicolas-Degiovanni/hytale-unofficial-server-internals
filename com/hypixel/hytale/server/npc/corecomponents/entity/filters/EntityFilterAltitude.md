---
description: Architectural reference for EntityFilterAltitude
---

# EntityFilterAltitude

**Package:** com.hypixel.hytale.server.npc.corecomponents.entity.filters
**Type:** Transient Configuration Object

## Definition
```java
// Signature
public class EntityFilterAltitude extends EntityFilterBase {
```

## Architecture & Concepts
The EntityFilterAltitude is a specific implementation of the **Strategy Pattern** used within the server-side NPC targeting and behavior system. It functions as a declarative predicate, allowing game designers to filter potential entity targets based on their vertical position relative to the ground.

This component acts as a bridge between the high-level NPC asset definitions and the low-level game state. It encapsulates a single, reusable filtering rule: "Is the target entity within a specific altitude range?". This enables complex targeting logic, such as distinguishing between flying and ground-based entities, to be defined in data rather than hard-coded.

Internally, it operates on the Entity-Component-System (ECS) paradigm. During evaluation, it queries the target entity for its NPCEntity component to access its MotionController, which provides the authoritative height-over-ground measurement.

## Lifecycle & Ownership
- **Creation:** This object is not intended for manual instantiation. It is constructed exclusively by the NPC asset loading pipeline via its corresponding builder, BuilderEntityFilterAltitude. This occurs when the server deserializes an NPC's behavior configuration from asset files.
- **Scope:** The lifetime of an EntityFilterAltitude instance is tightly coupled to the NPC behavior or role that defines it. It persists in memory as part of an NPC's loaded configuration.
- **Destruction:** The object has no explicit destruction method. It becomes eligible for garbage collection when its parent NPC configuration is unloaded, which typically happens when the NPC type is no longer active in the world or during a server-wide asset reload.

## Internal State & Concurrency
- **State:** Immutable. The core state is the `altitudeRange` array, which is a final field initialized at construction. The object holds no other mutable state and does not cache any world data. Each call to its evaluation method re-fetches live data from the entity store.
- **Thread Safety:** This class is inherently thread-safe. Its immutable nature ensures that it can be safely read by multiple threads. However, the `matchesEntity` method is expected to be called from the main server thread, as the `Store<EntityStore>` it receives is not guaranteed to be thread-safe for mutations. The filter itself introduces no concurrency hazards.

## API Surface
The public contract is minimal, focusing on the evaluation and cost-assessment of the filter.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| matchesEntity(ref, targetRef, role, store) | boolean | O(1) | Evaluates the target entity against the configured altitude range. Returns true if the entity's height over ground is within the inclusive range. Assumes component lookups are constant time. |
| cost() | int | O(1) | Returns the pre-defined computational cost for this filter. For EntityFilterAltitude, the cost is considered negligible and always returns zero. |

## Integration Patterns

### Standard Usage
This class is not invoked directly in gameplay code. It is configured within an NPC's asset files and executed by higher-level systems like a behavior tree or a target acquisition module. The following example demonstrates how a *system* would use an instance of this filter.

```java
// A hypothetical TargetAcquisitionSystem processing a list of potential targets
List<Ref<EntityStore>> potentialTargets = ...;
EntityFilterBase altitudeFilter = npc.getCurrentBehavior().getFilter(); // Filter is loaded from config

List<Ref<EntityStore>> validTargets = potentialTargets.stream()
    .filter(target -> altitudeFilter.matchesEntity(npcRef, target, npcRole, entityStore))
    .collect(Collectors.toList());

// The validTargets list now only contains entities that passed the altitude check.
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never create an instance using `new EntityFilterAltitude()`. The constructor requires a builder object that is only available within the asset loading context. Bypassing the asset pipeline will result in an unconfigured and non-functional filter.
- **Misinterpretation of Height:** This filter checks height *over the ground*, not absolute world Y-coordinate. Using it to check if an entity is above or below a specific world coordinate will produce incorrect results.

## Data Pipeline
EntityFilterAltitude acts as a gate in the data flow of an NPC's decision-making process. It reduces a set of potential targets to a smaller, valid set.

> Flow:
> Behavior Tree Node -> Target Acquisition System -> Full List of Nearby Entities -> **EntityFilterAltitude.matchesEntity()** -> Filtered List of Valid Targets -> Action Execution (e.g., MoveTo, Attack)

