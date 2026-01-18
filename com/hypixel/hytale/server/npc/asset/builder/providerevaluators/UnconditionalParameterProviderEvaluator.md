---
description: Architectural reference for UnconditionalParameterProviderEvaluator
---

# UnconditionalParameterProviderEvaluator

**Package:** com.hypixel.hytale.server.npc.asset.builder.providerevaluators
**Type:** Transient

## Definition
```java
// Signature
public class UnconditionalParameterProviderEvaluator implements ParameterProviderEvaluator {
```

## Architecture & Concepts
The UnconditionalParameterProviderEvaluator is a concrete, self-contained implementation of the ParameterProviderEvaluator interface. Its primary role within the NPC asset system is to represent a static, unchanging set of parameters.

The "Unconditional" nature is its defining characteristic. Unlike more complex evaluators that might query dynamic game state or entity properties, this class's logic is based entirely on the data provided at the moment of its creation. It answers the simple question: "Does this component statically declare parameter X of type Y?".

Architecturally, it serves as a leaf node in the asset resolution process. The intentional no-op implementation of the resolveReferences method signifies that it has no downstream dependencies on other assets or builders. This makes it a lightweight and highly performant component, ideal for defining the baseline, non-negotiable parameters of an NPC behavior or asset.

### Lifecycle & Ownership
-   **Creation:** Instantiated directly by higher-level asset parsers or builders when a static parameter block is encountered in an asset definition file. It is not managed by a service locator or dependency injection container.
-   **Scope:** The object's lifetime is strictly bound to its owning asset definition or builder. It is typically short-lived, existing only during the asset loading and validation phase.
-   **Destruction:** The object is reclaimed by the garbage collector once its owner is dereferenced. No explicit cleanup or disposal methods are required.

## Internal State & Concurrency
-   **State:** The internal state consists of a Map linking parameter names to their respective ParameterType. This state is populated exclusively within the constructor and is **effectively immutable** post-construction. The class does not cache or reference any external data.
-   **Thread Safety:** This class is **thread-safe**. All mutation is confined to the constructor, which is inherently single-threaded for that instance. All subsequent read operations via hasParameter are safe for concurrent access from multiple threads without requiring synchronization, as the internal map is never modified after initialization.

## API Surface
The public contract is minimal, focusing solely on parameter validation.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| hasParameter(String, ParameterType) | boolean | O(1) | Checks for the existence of a parameter with a matching name and type. |
| resolveReferences(BuilderManager) | void | O(1) | No-op. Fulfills the interface contract but performs no action. |

## Integration Patterns

### Standard Usage
This class is intended to be created and used internally by the asset building system to represent a fixed set of parameters defined in a configuration file.

```java
// Example: Inside an asset loader or builder
String[] definedParams = {"maxHealth", "movementSpeed"};
ParameterType[] definedTypes = {ParameterType.INTEGER, ParameterType.FLOAT};

// The evaluator is created to encapsulate this static definition
ParameterProviderEvaluator evaluator = new UnconditionalParameterProviderEvaluator(definedParams, definedTypes);

// A consumer can then query the evaluator to validate parameter availability
if (evaluator.hasParameter("movementSpeed", ParameterType.FLOAT)) {
    // Logic for components that require a movementSpeed parameter
}
```

### Anti-Patterns (Do NOT do this)
-   **Mismatched Constructor Arguments:** Providing arrays of different lengths for parameters and types to the constructor is a contract violation and will raise an immediate IllegalArgumentException. The system expects a one-to-one mapping.
-   **Dynamic State Representation:** Do not use this class to represent parameters that change based on game state. Its purpose is for static, compile-time definitions only. Using it for dynamic data would violate its "Unconditional" design principle.

## Data Pipeline
This component does not transform data in a pipeline. Instead, it acts as a data source or a predicate during the asset validation and construction phase.

> Flow:
> Asset File (JSON/HOCON) -> Asset Parser -> **UnconditionalParameterProviderEvaluator** (Instantiation) -> Behavior Logic Builder -> Parameter Query & Validation

