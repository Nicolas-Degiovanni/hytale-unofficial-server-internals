---
description: Architectural reference for BuilderEntityFilterAnd
---

# BuilderEntityFilterAnd

**Package:** com.hypixel.hytale.server.npc.corecomponents.entity.filters.builders
**Type:** Transient

## Definition
```java
// Signature
public class BuilderEntityFilterAnd extends BuilderEntityFilterMany {
```

## Architecture & Concepts
The BuilderEntityFilterAnd class is a factory component within the server's NPC asset loading pipeline. It embodies the Builder design pattern, specifically designed to translate a declarative asset definition into a concrete, runtime filter object.

Its primary function is to construct an EntityFilterAnd instance, which represents a composite filter that evaluates to true only if **all** of its child filters also evaluate to true. This class does not perform any entity filtering itself; it is a configuration-time object responsible for assembling the runtime logic from a list of other filter definitions.

This builder acts as a node in the asset deserialization tree. When the asset loader encounters a definition for a logical AND filter, it instantiates this class, populates it with the definitions of the child filters, and then invokes the build method to produce the final, executable IEntityFilter object.

## Lifecycle & Ownership
- **Creation:** Instantiated exclusively by the server's asset loading framework during the parsing of NPC definition files (e.g., JSON or HOCON). A higher-level factory or deserializer creates an instance when it identifies a component type corresponding to this builder.
- **Scope:** Extremely short-lived and transient. An instance of BuilderEntityFilterAnd exists only for the duration of its corresponding asset block being parsed.
- **Destruction:** The builder instance is immediately eligible for garbage collection after the build method has been called and its result (the EntityFilterAnd object) has been integrated into the parent NPC configuration. The object it *creates* has a much longer lifecycle, tied to the NPC that uses the filter.

## Internal State & Concurrency
- **State:** This class is stateful. It internally manages the list of child filter builders via the inherited objectListHelper field. This state is populated once during asset deserialization and is treated as immutable thereafter.
- **Thread Safety:** This class is **not thread-safe** and must not be accessed concurrently. It is designed to operate within a strictly single-threaded asset loading pipeline to guarantee deterministic and safe construction of the final object graph.

## API Surface
The public API is minimal, focusing entirely on metadata and the core build operation.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| build(BuilderSupport) | IEntityFilter | O(N) | Constructs the final EntityFilterAnd object. N is the number of child filters. Returns null if the list of child filters is empty. |
| getShortDescription() | String | O(1) | Provides a human-readable summary for tooling and debugging. |
| getLongDescription() | String | O(1) | Provides a detailed description, currently identical to the short one. |
| getBuilderDescriptorState() | BuilderDescriptorState | O(1) | Returns metadata indicating the production-readiness of this builder. |

## Integration Patterns

### Standard Usage
This class is not intended for direct use by game logic developers. It is invoked transparently by the asset loading system. The conceptual usage by the system is as follows.

```java
// PSEUDOCODE: How the asset system uses this builder
// 1. Asset loader parses a JSON block and creates the builder
BuilderEntityFilterAnd builder = assetLoader.createBuilderFrom("hytale:and_filter");
builder.setChildFilters(parsedChildFilterDefinitions);

// 2. The loader invokes build to get the runtime object
BuilderSupport support = assetLoader.getSupportContext();
IEntityFilter runtimeFilter = builder.build(support);

// 3. The runtimeFilter is attached to an NPC's behavior component
npc.getBehaviorTree().addFilter(runtimeFilter);
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never call new BuilderEntityFilterAnd(). The asset system is solely responsible for its creation and configuration. Manually creating this object bypasses the necessary asset pipeline context.
- **Runtime Invocation:** This builder and its methods must never be called from the main game loop or any runtime game systems. It is a pre-computation component, not a gameplay one.
- **State Mutation:** Do not attempt to modify the builder's state after it has been configured by the asset loader. Calling build multiple times on the same instance may lead to unpredictable behavior.

## Data Pipeline
This builder serves as a transformation step, converting declarative data into an executable object.

> Flow:
> NPC Asset File (e.g., JSON) -> Asset Deserializer -> **BuilderEntityFilterAnd** -> EntityFilterAnd (Instance) -> NPC Behavior Tree

