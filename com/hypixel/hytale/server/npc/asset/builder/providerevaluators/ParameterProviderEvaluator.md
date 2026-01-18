---
description: Architectural reference for ParameterProviderEvaluator
---

# ParameterProviderEvaluator

**Package:** com.hypixel.hytale.server.npc.asset.builder.providerevaluators
**Type:** Contract Interface

## Definition
```java
// Signature
public interface ParameterProviderEvaluator extends ProviderEvaluator {
```

## Architecture & Concepts
The ParameterProviderEvaluator interface defines a specialized contract for components within the server-side NPC asset generation system. It extends the base ProviderEvaluator, adding a critical capability: the ability to query whether a provider supports a specific, named parameter.

This interface embodies the **Specification Pattern**. Its primary role is to allow the NPC asset builder to introspect a data provider's capabilities *before* attempting to evaluate it. This prevents runtime errors and enables the construction of complex, data-driven NPCs where different providers can satisfy different parts of an asset's definition.

It acts as a gatekeeper in the provider selection logic, decoupling the core asset builder from the concrete implementations of various data providers. This allows for a highly extensible system where new providers with unique parameters can be introduced without modifying the central asset construction logic.

### Lifecycle & Ownership
As an interface, ParameterProviderEvaluator does not have a lifecycle of its own. The lifecycle described here pertains to its concrete implementations.

-   **Creation:** Implementations are instantiated by the NPC asset building framework, often during the initialization of the asset loading pipeline. They are typically registered with or passed to a central builder or factory responsible for NPC generation.
-   **Scope:** The lifetime of an implementation is tied to the asset building process it serves. It may be transient (created for a single evaluation) or scoped to the lifetime of a parent AssetBuilder service.
-   **Destruction:** Implementations are subject to standard Java garbage collection. They are eligible for cleanup once the asset building process that holds a reference to them is complete and no other references remain.

## Internal State & Concurrency
-   **State:** The interface itself is stateless. Implementations are **strongly recommended** to be stateless. Their evaluation should depend solely on the input arguments and the configuration of the provider they represent, not on mutable internal state.
-   **Thread Safety:** Implementations must be thread-safe. The NPC asset system may perform asset generation in parallel to improve server startup times or handle dynamic NPC spawning. Any implementation of this interface must be safely callable from multiple threads without causing race conditions or data corruption. This is typically achieved by making them stateless.

## API Surface
The public contract is focused on querying provider capabilities.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| hasParameter(String, ParameterType) | boolean | O(1) | Checks if the underlying provider supports a parameter with the given name and type. Returns true if supported, false otherwise. |

## Integration Patterns

### Standard Usage
This interface is not meant to be invoked directly by most game logic. It is a service provider interface used by the NPC asset building system. A builder will use it to filter and select an appropriate data provider from a collection.

```java
// Hypothetical usage within an asset builder
List<ProviderEvaluator> evaluators = getAvailableEvaluators();
String requiredParam = "spawn_condition";
ParameterType requiredType = ParameterType.BOOLEAN;

for (ProviderEvaluator eval : evaluators) {
    if (eval instanceof ParameterProviderEvaluator) {
        ParameterProviderEvaluator ppe = (ParameterProviderEvaluator) eval;
        if (ppe.hasParameter(requiredParam, requiredType)) {
            // This provider is suitable, proceed with evaluation
            return eval.evaluate();
        }
    }
}
```

### Anti-Patterns (Do NOT do this)
-   **Assuming Implementation:** Do not cast to a concrete implementation of this interface. The system is designed to work with the abstract contract.
-   **Ignoring The Check:** Do not attempt to retrieve a parameter from a provider without first calling hasParameter. This defeats the purpose of the interface and can lead to runtime exceptions if the parameter does not exist.

## Data Pipeline
This component is part of the *control flow*, not the data flow. It makes a decision that directs the flow of data rather than transforming the data itself.

> Flow:
> NPC Asset Definition -> Asset Builder -> **ParameterProviderEvaluator.hasParameter()** -> [Decision Logic] -> Select & Invoke Provider -> Value Generation

