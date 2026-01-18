---
description: Architectural reference for FeatureProviderEvaluator
---

# FeatureProviderEvaluator

**Package:** com.hypixel.hytale.server.npc.asset.builder.providerevaluators
**Type:** Contract Interface

## Definition
```java
// Signature
public interface FeatureProviderEvaluator extends ProviderEvaluator {
```

## Architecture & Concepts
The FeatureProviderEvaluator interface defines a critical contract within the server-side NPC asset generation pipeline. It acts as a specialized predicate, extending the base ProviderEvaluator to enable a query-based selection mechanism for asset providers.

In the Hytale engine, NPCs are constructed dynamically from a collection of features such as appearance, equipment, and animations. The system that assembles these assets, the Asset Builder, must select appropriate data providers for each required feature. This interface is the mechanism for that selection.

An implementing class is responsible for declaring whether it can supply a given set of features, represented by the Feature enum. This allows the Asset Builder to iterate through a registry of potential providers and filter them down to only those that can satisfy the requirements of the NPC being generated. This implements a **Strategy** or **Specification** pattern, decoupling the asset assembly logic from the concrete data providers.

## Lifecycle & Ownership
As an interface, FeatureProviderEvaluator does not have a lifecycle of its own. The lifecycle described here pertains to the concrete classes that implement this contract.

- **Creation:** Implementations are expected to be instantiated and registered with the NPC asset building system during server initialization. They are typically lightweight, stateless objects.
- **Scope:** An instance of an implementing class should be considered a session-scoped singleton. It persists for the lifetime of the server process to be available for any NPC generation request.
- **Destruction:** Managed by the server's main lifecycle. No manual destruction is required.

**WARNING:** Implementations of this interface must be designed as stateless services. Storing request-specific state within an implementation is a severe anti-pattern that will lead to race conditions and incorrect NPC generation.

## Internal State & Concurrency
- **State:** The contract implies that implementations should be **stateless and immutable**. Their purpose is to evaluate input against a fixed set of capabilities, not to store or modify data.
- **Thread Safety:** All implementations of this interface **must be thread-safe**. The NPC generation system may be invoked concurrently from multiple threads to build different NPCs simultaneously. A stateless implementation is inherently thread-safe.

## API Surface
The public contract consists of a single method inherited from the base interface, specialized for NPC features.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| provides(EnumSet<Feature> features) | boolean | O(N) | Evaluates if the underlying provider can supply all features specified in the input set. Returns true if all features are supported, false otherwise. |

## Integration Patterns

### Standard Usage
The primary integration pattern is for the NPC Asset Builder to query a collection of registered evaluators to find a suitable provider for a specific generation task.

```java
// How the NPC Asset Builder uses this interface
EnumSet<Feature> requiredFeatures = EnumSet.of(Feature.HAIR_STYLE_A, Feature.ARMOR_SET_IRON);
FeatureProviderEvaluator selectedEvaluator = null;

for (FeatureProviderEvaluator evaluator : registeredEvaluators) {
    if (evaluator.provides(requiredFeatures)) {
        selectedEvaluator = evaluator;
        break;
    }
}

if (selectedEvaluator != null) {
    // Proceed with asset generation using the provider associated with the evaluator
}
```

### Anti-Patterns (Do NOT do this)
- **Stateful Implementations:** Do not store any per-request or mutable state in a class that implements this interface. This will break in a multi-threaded environment.
- **Slow Evaluation:** The provides method must execute quickly and without I/O. It is a frequent, high-throughput check. Performing database lookups, file reads, or network calls within this method will severely degrade server performance.
- **Partial Matches:** The contract implies an "all or nothing" check. An implementation should not return true if it can only provide a subset of the requested features.

## Data Pipeline
This interface is a control-flow component, not a data-flow component. It directs the flow of the asset generation process by acting as a gate.

> Flow:
> NPC Generation Request -> Asset Builder receives required **EnumSet<Feature>** -> **FeatureProviderEvaluator.provides()** check -> Suitable Asset Provider selected -> Provider streams asset data to Builder

