---
description: Architectural reference for EntityFilterOr
---

# EntityFilterOr

**Package:** com.hypixel.hytale.server.npc.corecomponents.entity.filters
**Type:** Transient

## Definition
```java
// Signature
public class EntityFilterOr extends EntityFilterMany {
```

## Architecture & Concepts
The EntityFilterOr class is an implementation of the **Composite** design pattern. It functions as a container component that aggregates multiple IEntityFilter instances and evaluates them with a logical OR operation.

This component is a fundamental building block within the server-side NPC AI and behavior systems. It enables the creation of complex, hierarchical targeting and filtering rules from simpler, single-purpose filters. For example, an NPC's behavior tree might use an EntityFilterOr to define a targeting condition such as "attack any entity that is a Player OR is tagged as Hostile".

By composing filters in this manner, the system achieves significant flexibility and reusability, allowing designers to define sophisticated NPC behaviors declaratively in configuration files without writing new code.

### Lifecycle & Ownership
- **Creation:** EntityFilterOr instances are not intended for direct instantiation within imperative game logic. They are almost exclusively created by a deserialization process that parses NPC behavior profiles from asset files (e.g., JSON or HOCON). A factory or builder responsible for constructing an NPC's behavior tree will instantiate this class based on the defined configuration.
- **Scope:** The lifetime of an EntityFilterOr object is bound to its parent component, typically a specific behavior, role, or state within an NPC's AI graph. It is considered a short-lived configuration object.
- **Destruction:** The object is managed by the Java garbage collector. It becomes eligible for collection as soon as the owning NPC behavior configuration is unloaded or replaced. No explicit cleanup is required.

## Internal State & Concurrency
- **State:** **Effectively Immutable**. The internal list of child filters is provided during construction and is not modified during the object's lifetime. The class itself holds no other mutable state.
- **Thread Safety:** **Conditionally Thread-Safe**. The class contains no internal synchronization mechanisms. Its thread safety is entirely dependent on the thread safety of the IEntityFilter instances it holds. Within the standard server architecture, entity evaluation is confined to a single world-tick thread, making concurrent access an edge case.

**WARNING:** If custom systems attempt to evaluate entity filters from multiple threads, the contained filters must be verified for thread safety.

## API Surface
The public contract is minimal, focused entirely on the evaluation logic inherited from the IEntityFilter interface.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| matchesEntity(ref, targetRef, role, store) | boolean | O(N) | Iterates through the contained filters and returns true upon the first successful match. Returns false if no filters match. The complexity is O(1) in the best case (the first filter matches). |

## Integration Patterns

### Standard Usage
This class is not used directly but is defined declaratively as part of a larger NPC behavior configuration. The system's AI engine will then invoke it during its evaluation ticks.

A conceptual configuration might look like this:

```yaml
# Example NPC Behavior Asset
targetFilter: {
  # The system would deserialize this into an EntityFilterOr instance
  type: "or",
  filters: [
    { type: "isPlayer" },
    { type: "hasTag", tag: "Hostile" }
  ]
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Avoid `new EntityFilterOr(...)` in dynamic game logic. This pattern violates the principle of separating configuration (data) from execution (code) and leads to brittle, hard-to-maintain AI logic.
- **Empty Filter List:** Constructing an EntityFilterOr with an empty list of child filters is valid but logically useless, as it will always evaluate to false. This typically indicates a misconfiguration in an asset file.
- **Complex Nesting:** While powerful, deeply nested EntityFilterOr and EntityFilterAnd components can become difficult to debug and reason about. Favor flatter filter hierarchies where possible.

## Data Pipeline
EntityFilterOr acts as a decision node within the NPC entity evaluation pipeline. It does not transform data but rather returns a boolean predicate that controls the flow of a targeting or interaction check.

> Flow:
> NPC Behavior Tree Tick -> Target Selection System -> Iterate Potential Entities -> **EntityFilterOr.matchesEntity()** -> Return boolean -> Target Accepted/Rejected


