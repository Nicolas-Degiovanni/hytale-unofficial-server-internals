---
description: Architectural reference for EntityFilterHeightDifference
---

# EntityFilterHeightDifference

**Package:** com.hypixel.hytale.server.npc.corecomponents.entity.filters
**Type:** Transient

## Definition
```java
// Signature
public class EntityFilterHeightDifference extends EntityFilterBase {
```

## Architecture & Concepts
The EntityFilterHeightDifference is a server-side component that functions as a specific predicate within the NPC AI's target selection framework. It is a concrete implementation of the *Strategy Pattern*, where it provides one of many possible filtering algorithms defined by the EntityFilterBase contract.

Its primary purpose is to determine if a target entity is within a specified vertical distance range relative to the querying NPC. This allows for the creation of more realistic and context-aware behaviors. For example, an NPC can be configured to ignore targets that are on a floor far above or below it, preventing illogical pathfinding or combat engagement.

This filter integrates directly with the server's Entity-Component-System (ECS). During its evaluation, it queries the world's EntityStore for essential components like TransformComponent, BoundingBox, and ModelComponent to access positional and physical data. The static COST field indicates its participation in a query optimization system, which likely executes lower-cost filters before more computationally expensive ones to fail-fast and reduce server load.

## Lifecycle & Ownership
- **Creation:** Instances are not created directly. They are instantiated by the server's asset loading pipeline when parsing NPC behavior configurations. A corresponding builder, BuilderEntityFilterHeightDifference, reads parameters from an asset file (e.g., a JSON definition) and constructs the filter object.
- **Scope:** The object's lifetime is bound to the specific NPC behavior or role that defines it. It is a stateless, reusable configuration object that persists as long as its parent AI configuration is loaded in memory.
- **Destruction:** The object is managed by the Java garbage collector. It is eligible for collection once the owning NPC configuration is unloaded, for instance, when a zone unloads or a game mode ends. There is no manual destruction method.

## Internal State & Concurrency
- **State:** Immutable. The core parameters of the filter—minHeightDifference, maxHeightDifference, and useEyePosition—are declared final and are initialized only once during construction. The object holds no mutable state and does not cache any data between calls.

- **Thread Safety:** This class is inherently thread-safe. Its immutability guarantees that it can be safely shared and executed by multiple NPC agents across different worker threads without locks or synchronization. All state required for its computation is passed into the matchesEntity method, ensuring no side effects.

## API Surface
The public contract is minimal and focused exclusively on the filtering operation.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| matchesEntity(ref, targetRef, role, store) | boolean | O(1) | Evaluates if the target entity meets the height criteria. Involves a fixed number of component lookups and calculations. Returns false if the target lacks required components like BoundingBox. |
| cost() | int | O(1) | Returns the static computational cost (200) of this filter for query planning purposes. |

## Integration Patterns

### Standard Usage
This filter is not intended for direct invocation by game logic developers. It is automatically managed and executed by higher-level AI systems, such as a behavior tree's target selection node. The system iterates through a list of potential targets and applies this filter.

```java
// Hypothetical usage within an AI Target Selector service
// Note: This is a conceptual example. Developers do not write this code.

List<EntityFilterBase> filters = npc.getBehavior().getTargetFilters();
Ref<EntityStore> potentialTarget = findNearbyEntity();

boolean isValidTarget = true;
for (EntityFilterBase filter : filters) {
    // The system invokes matchesEntity on this and other filters
    if (!filter.matchesEntity(npc.getRef(), potentialTarget, npc.getRole(), world.getStore())) {
        isValidTarget = false;
        break;
    }
}

if (isValidTarget) {
    npc.setTarget(potentialTarget);
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never create an instance using `new EntityFilterHeightDifference()`. The object's parameters must be supplied by the asset configuration pipeline via its corresponding builder. Direct creation bypasses the intended data-driven design.
- **Ignoring The Cost System:** When designing custom AI systems, do not invoke filters in a random order. The cost method provides a critical hint for performance. High-cost filters should be executed last in a chain of logical AND conditions.

## Data Pipeline
EntityFilterHeightDifference acts as a conditional gate in a data flow. It does not transform data but rather validates it, producing a boolean outcome that determines the subsequent flow of logic.

> Flow:
> AI Behavior Tick -> Target Acquisition System -> For each potential entity -> **EntityFilterHeightDifference.matchesEntity** -> Boolean Pass/Fail -> Target List Pruning -> Final Target Selection

