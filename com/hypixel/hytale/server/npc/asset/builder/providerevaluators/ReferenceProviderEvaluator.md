---
description: Architectural reference for ReferenceProviderEvaluator
---

# ReferenceProviderEvaluator

**Package:** com.hypixel.hytale.server.npc.asset.builder.providerevaluators
**Type:** Transient

## Definition
```java
// Signature
public class ReferenceProviderEvaluator implements FeatureProviderEvaluator, ParameterProviderEvaluator {
```

## Architecture & Concepts
The ReferenceProviderEvaluator is a specialized component within the NPC asset building framework that functions as a **lazy-loading proxy** or **forwarding reference**. Its primary role is not to provide features or parameters directly, but to point to another asset Builder that does. This enables powerful asset composition, allowing designers to define a common set of features (e.g., a "base guard" template) and have other NPC definitions reference and inherit from it without duplicating data.

This class embodies the **Proxy Pattern**. At construction, it is an inert placeholder, containing only an index and a type that identify the target Builder. It remains in this unresolved state until the system explicitly links all asset definitions. After this resolution phase, it delegates all calls for providing features or parameters to the actual set of providers from the referenced Builder.

This mechanism is critical for managing dependencies and load order within the asset system. It allows asset files to be parsed in any order, with the connections between them being established in a distinct, subsequent "linking" step managed by the BuilderManager.

## Lifecycle & Ownership
The lifecycle of a ReferenceProviderEvaluator is tightly coupled to the asset loading and linking process. It exists in two distinct states: *unresolved* and *resolved*.

-   **Creation:** An instance is created by the asset parsing system when it encounters a reference directive within an NPC definition file. It is immediately added to a parent Builder's collection of evaluators in an unresolved state.

-   **Scope:** Its lifetime is bound to its parent Builder object. It is effectively a component of a larger asset definition.

-   **Resolution (State Transition):** The pivotal lifecycle event is the invocation of its `resolveReferences` method. This is orchestrated exclusively by the BuilderManager during a global linking phase after all asset files have been parsed. This call populates the internal `resolvedProviderSet`, transitioning the object to its *resolved* state.

-   **Destruction:** The object is eligible for garbage collection when its parent Builder is unloaded or discarded. There is no explicit destruction method.

## Internal State & Concurrency
-   **State:** The internal state is mutable. The `resolvedProviderSet` field is initialized to null and is populated exactly once during the `resolveReferences` call. This represents a one-way state transition from unresolved to resolved.

-   **Thread Safety:** This class is **not thread-safe** and must be operated within a single-threaded context, particularly during the resolution phase. The design assumes a strict, phased execution model common in game engines:
    1.  **Parse Phase:** All assets are loaded and evaluators are created.
    2.  **Link Phase:** `resolveReferences` is called on all evaluators.
    3.  **Usage Phase:** The resolved evaluators are used to build the final NPC assets.

    Concurrent access could lead to race conditions, where one thread attempts to use the evaluator via `provides` while another is in the middle of the `resolveReferences` call, likely resulting in a NullPointerException.

## API Surface
The public contract is designed for interaction by the BuilderManager and the parent Builder.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| provides(EnumSet) | boolean | O(N) | Delegates the check to the resolved providers. Returns false if unresolved. N is the number of providers in the referenced set. |
| hasParameter(String, Type) | boolean | O(N) | Delegates the check to the resolved providers. Returns false if unresolved. N is the number of providers in the referenced set. |
| resolveReferences(BuilderManager) | void | O(1) | **CRITICAL:** Populates the internal provider set by looking up the referenced Builder in the manager's cache. This is the state transition trigger. |

## Integration Patterns

### Standard Usage
The evaluator is intended to be managed entirely by the asset building system. A developer would typically never interact with this class directly. The system-level flow is as follows:

```java
// 1. During a system-wide linking phase, the BuilderManager resolves all references.
// This is a conceptual example of the orchestration logic.

for (Builder<?> builder : allParsedBuilders) {
    FeatureEvaluatorHelper helper = builder.getEvaluatorHelper();
    for (ProviderEvaluator evaluator : helper.getProviders()) {
        // The manager identifies evaluators that need resolving
        if (evaluator instanceof ReferenceProviderEvaluator) {
            ((ReferenceProviderEvaluator) evaluator).resolveReferences(builderManager);
        }
    }
}

// 2. Later, when a builder needs to check for a feature, it can now query
// the resolved reference evaluator transparently.
boolean hasFeature = someBuilder.getEvaluatorHelper().provides(featureSet);
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Never create an instance using `new ReferenceProviderEvaluator()`. These objects must be generated by the asset parser.
-   **Premature Access:** Calling `provides` or `hasParameter` before the global `resolveReferences` phase has completed. This will always return `false` and can lead to silent, difficult-to-diagnose failures where features appear to be missing.
-   **Re-Resolving:** Calling `resolveReferences` more than once is not explicitly forbidden but is outside the intended lifecycle and has no effect.

## Data Pipeline
The ReferenceProviderEvaluator acts as a gate in the data flow, which is opened during the linking phase.

> **Flow (Before Resolution):**
> Feature Request -> Builder -> **ReferenceProviderEvaluator** -> `false` (dead end)

> **Flow (After Resolution):**
> `BuilderManager.resolveAll()` -> **ReferenceProviderEvaluator.resolveReferences()** -> Internal state populated
>
> Feature Request -> Builder -> **ReferenceProviderEvaluator** -> (Delegation) -> Referenced Provider Set -> Boolean Result

