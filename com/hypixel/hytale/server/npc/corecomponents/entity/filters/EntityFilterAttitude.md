---
description: Architectural reference for EntityFilterAttitude
---

# EntityFilterAttitude

**Package:** com.hypixel.hytale.server.npc.corecomponents.entity.filters
**Type:** Transient

## Definition
```java
// Signature
public class EntityFilterAttitude extends EntityFilterBase {
```

## Architecture & Concepts
The EntityFilterAttitude is a concrete implementation of the Strategy pattern, designed to operate within the server's NPC AI and behavior systems. It serves as a specific, reusable filtering rule that determines whether an entity is a valid candidate for interaction based on the established relationship, or *Attitude*, between the NPC and the target.

This component is a fundamental building block for creating complex AI targeting logic. It allows designers to declaratively specify which types of entities an NPC should consider hostile, friendly, neutral, or other custom relationships. For example, a guard NPC might use this filter to select only entities with a **Hostile** attitude for its attack behavior.

Its design is data-driven, typically being configured and instantiated by the asset loading pipeline from JSON or HOCON definitions via its corresponding builder, BuilderEntityFilterAttitude.

## Lifecycle & Ownership
- **Creation:** Primarily instantiated by the NPC asset loading system during the parsing of an NPC's behavior configuration. It can also be created programmatically at runtime by supplying an array of Attitude enums, a pattern useful for dynamic behaviors or testing.
- **Scope:** The lifetime of an EntityFilterAttitude instance is tied to the specific AI behavior or task that owns it, such as a targeting selector or an ability condition. It is not a global or session-scoped object.
- **Destruction:** The object is managed by the Java Garbage Collector. It holds no native resources or persistent state and is cleaned up when the owning AI behavior is discarded.

## Internal State & Concurrency
- **State:** Immutable. The core state is the `attitudes` EnumSet, which is populated exclusively during construction and is never modified thereafter. This makes the filter's configuration fixed for its entire lifetime.
- **Thread Safety:** This class is inherently thread-safe due to its immutable state. The primary method, `matchesEntity`, is a pure function with no side effects on the filter's internal state. It can be safely invoked by multiple threads concurrently.

    **WARNING:** While the filter itself is thread-safe, the `Store<EntityStore>` and `Role` objects passed into its methods may not be. Callers are responsible for ensuring that access to the world state is properly synchronized.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| matchesEntity(ref, targetRef, role, store) | boolean | O(1) | The core filtering logic. Returns true if the attitude of the source entity towards the target is contained within the filter's configured set of attitudes. |
| cost() | int | O(1) | Returns a constant performance cost of 0, indicating this is a computationally inexpensive filter. |
| registerWithSupport(role) | void | O(1) | A critical setup hook. Declares to the world system that this filter requires the AttitudeCache to be active for the associated NPC. |

## Integration Patterns

### Standard Usage
This filter is not used in isolation. It is typically part of a chain or collection of EntityFilterBase implementations that are evaluated sequentially by a higher-level AI system to narrow down a set of potential targets.

```java
// Pseudo-code demonstrating a targeting system using the filter
List<EntityFilterBase> filters = behavior.getFilters();
Ref<EntityStore> self = ...;
Ref<EntityStore> potentialTarget = ...;
Role role = ...;
Store<EntityStore> store = ...;

boolean isTargetValid = true;
for (EntityFilterBase filter : filters) {
    // The filter chain short-circuits on the first failure
    if (!filter.matchesEntity(self, potentialTarget, role, store)) {
        isTargetValid = false;
        break;
    }
}
```

### Anti-Patterns (Do NOT do this)
- **Failure to Register:** Neglecting to call `registerWithSupport` during the NPC's initialization phase is a critical error. This will prevent the underlying `AttitudeCache` from being enabled, leading to severe performance degradation or potential runtime exceptions when `matchesEntity` is invoked.
- **Stateful Logic:** Do not extend this class to add mutable state. The filtering system relies on these components being stateless and reusable. Stateful logic should be managed by the owning `Role` or AI behavior.

## Data Pipeline
The filter acts as a gate in the data flow of an AI's decision-making process. It queries an external system (WorldSupport) to enrich the data about a potential target before making a boolean decision.

> Flow:
> AI Targeting System -> Provides (NPC, Potential Target) -> **EntityFilterAttitude.matchesEntity** -> Queries WorldSupport for Attitude -> **EntityFilterAttitude** compares result against its internal EnumSet -> Returns Boolean Decision -> AI System Accepts/Rejects Target

