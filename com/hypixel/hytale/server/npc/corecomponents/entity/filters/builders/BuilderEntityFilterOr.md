---
description: Architectural reference for BuilderEntityFilterOr
---

# BuilderEntityFilterOr

**Package:** com.hypixel.hytale.server.npc.corecomponents.entity.filters.builders
**Type:** Transient

## Definition
```java
// Signature
public class BuilderEntityFilterOr extends BuilderEntityFilterMany {
```

## Architecture & Concepts
The BuilderEntityFilterOr is a factory component within the server-side NPC asset system. Its sole responsibility is to translate a data-driven configuration into a concrete runtime instance of an EntityFilterOr. This class embodies the Builder pattern, abstracting the complex construction of a composite logical filter.

In the Hytale NPC architecture, entity filters are used by AI behaviors to select or ignore potential targets. The BuilderEntityFilterOr constructs a composite filter that implements a logical OR operation. An entity will pass this filter if it passes *any* of the sub-filters contained within the list.

This builder is a critical link between the static asset definitions (e.g., JSON files describing an NPC's behavior) and the live, in-game AI engine. It ensures that complex logical conditions can be defined by designers without requiring direct code changes.

### Lifecycle & Ownership
- **Creation:** Instantiated dynamically by the NPC asset loading pipeline when it encounters an "or" filter definition in an asset file. It is not intended for manual creation by developers.
- **Scope:** Ephemeral. The builder's lifetime is confined to a single method call within the asset loading process. It exists only to produce one EntityFilterOr instance.
- **Destruction:** The object is eligible for garbage collection immediately after the `build` method returns. It holds no references and maintains no state post-construction.

## Internal State & Concurrency
- **State:** The BuilderEntityFilterOr is effectively stateless. The list of child filters it needs to construct is managed by its parent class, BuilderEntityFilterMany, via an internal helper object. This state is configured once by the asset parser before the build process begins.
- **Thread Safety:** This class is **not thread-safe**. The entire NPC asset building process is designed to be a single-threaded, synchronous operation. Concurrent calls to the `build` method on a single instance will result in undefined behavior and potential corruption of the resulting filter object.

## API Surface
The public API is minimal, designed for consumption by the automated asset system.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| build(BuilderSupport) | IEntityFilter | O(N) | Constructs an EntityFilterOr instance from its configured sub-filters. N is the number of sub-filters. Returns null if the sub-filter list is empty. |
| getShortDescription() | String | O(1) | Provides a brief, human-readable description for tooling and logs. |
| getBuilderDescriptorState() | BuilderDescriptorState | O(1) | Returns the stability level of this component, indicating its readiness for production use. |

## Integration Patterns

### Standard Usage
This class is not intended for direct use. Developers define the filter declaratively in an asset file. The system then uses this builder to realize the definition. The *output* of the builder is then used by the AI system.

```java
// PSEUDOCODE: How the AI system might use the *result* of the builder

// The filter is retrieved from a fully-constructed NPC asset
IEntityFilter orFilter = npc.getBehavior().getTargetingFilter();

// The filter is then used to test potential targets
if (orFilter.evaluate(potentialTarget)) {
    // This target matches at least one of the conditions in the OR block
    npc.setTarget(potentialTarget);
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never call `new BuilderEntityFilterOr()`. The asset pipeline is responsible for creating and configuring builder instances. Manual creation bypasses critical configuration steps.
- **Re-use:** Do not retain an instance of this builder for later use. It is designed to be single-use and should be discarded after the `build` method is called.

## Data Pipeline
The builder acts as a transformation stage in the data flow from asset definition to a usable runtime object.

> Flow:
> NPC Asset File (JSON) -> Asset Parsing System -> **BuilderEntityFilterOr** -> EntityFilterOr (In-Memory Object) -> NPC Behavior Tree Evaluation

