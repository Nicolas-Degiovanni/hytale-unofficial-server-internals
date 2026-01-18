---
description: Architectural reference for UnconditionalFeatureProviderEvaluator
---

# UnconditionalFeatureProviderEvaluator

**Package:** com.hypixel.hytale.server.npc.asset.builder.providerevaluators
**Type:** Transient

## Definition
```java
// Signature
public class UnconditionalFeatureProviderEvaluator implements FeatureProviderEvaluator {
```

## Architecture & Concepts
The UnconditionalFeatureProviderEvaluator is a concrete implementation of the FeatureProviderEvaluator strategy interface. It represents the simplest and most fundamental rule within the NPC asset generation system: the unconditional requirement of a single, specific Feature.

Its primary role is to act as a terminal predicate in an evaluation chain. When the asset system needs to determine if a component is suitable for an NPC, it queries a set of evaluators. This class answers the atomic question: "Does the set of requested features contain this *one* specific feature I was configured with?"

The "Unconditional" nature is a key design aspect. Its logic is static, context-free, and has no external dependencies. This is explicitly demonstrated by its empty implementation of the resolveReferences method, signaling to the BuilderManager that it is self-contained and requires no further processing or linking. This makes it a highly predictable and performant building block for more complex rule compositions.

### Lifecycle & Ownership
-   **Creation:** Instantiated directly by higher-level components, such as an asset parser or a composite provider, when an asset definition specifies a non-negotiable feature requirement. It is not managed by a service locator or dependency injection framework.
-   **Scope:** Short-lived. Its lifetime is strictly bound to its owning provider or asset definition. It is created on-demand during the asset loading and evaluation phase.
-   **Destruction:** Becomes eligible for garbage collection as soon as its parent object is dereferenced. No explicit cleanup or destruction logic is necessary.

## Internal State & Concurrency
-   **State:** **Immutable**. The internal state, consisting of the target Feature enum, is established at construction and assigned to a final field. It cannot be modified for the lifetime of the object.
-   **Thread Safety:** **Inherently thread-safe**. As an immutable object, an instance can be safely shared, published, and accessed by multiple threads concurrently without any external synchronization or locking mechanisms.

## API Surface
The public contract is minimal and focused on evaluation, adhering to the FeatureProviderEvaluator interface.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| provides(EnumSet<Feature> feature) | boolean | O(1) | Evaluates if the input set contains the single Feature this instance was configured with. This is a constant-time operation for EnumSet. |
| resolveReferences(BuilderManager manager) | void | O(1) | No-op. Fulfills the interface contract but performs no action, as this evaluator has no external dependencies to resolve. |

## Integration Patterns

### Standard Usage
This evaluator is typically used as part of a collection of rules within a larger provider. The system iterates through these evaluators to validate if an asset meets a set of required features.

```java
// Example: An asset provider holds a list of evaluators to define its requirements.
List<FeatureProviderEvaluator> requirements = new ArrayList<>();
requirements.add(new UnconditionalFeatureProviderEvaluator(Feature.WEARS_ARMOR));
requirements.add(new UnconditionalFeatureProviderEvaluator(Feature.HAS_SWORD));

// During NPC generation, a set of desired features is provided.
EnumSet<Feature> desiredFeatures = EnumSet.of(Feature.WEARS_ARMOR, Feature.IS_HOSTILE);

// The system checks each requirement.
boolean wearsArmor = requirements.get(0).provides(desiredFeatures); // returns true
boolean hasSword = requirements.get(1).provides(desiredFeatures); // returns false
```

### Anti-Patterns (Do NOT do this)
-   **Misuse for Complex Logic:** Do not attempt to extend this class to handle multi-feature logic (e.g., "provides FeatureA *or* FeatureB"). This class is intentionally designed for a single, atomic check. For complex rules, use a composite evaluator that combines multiple instances of this and other evaluators.
-   **Null Feature Construction:** The constructor is annotated with Nonnull. Passing a null Feature during instantiation is a contract violation and will lead to unpredictable behavior or NullPointerExceptions downstream.

## Data Pipeline
This class acts as a predicate or filter within the broader NPC asset selection pipeline. It does not transform data but rather gates its flow based on a boolean check.

> Flow:
> NPC Asset Definition Parsing -> **UnconditionalFeatureProviderEvaluator Instantiation** -> NPC Generation Request -> Feature Set Evaluation via provides() -> Boolean Result (Allow/Deny) -> Final Asset Selection

