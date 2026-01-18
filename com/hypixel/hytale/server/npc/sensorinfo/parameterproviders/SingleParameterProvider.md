---
description: Architectural reference for SingleParameterProvider
---

# SingleParameterProvider

**Package:** com.hypixel.hytale.server.npc.sensorinfo.parameterproviders
**Type:** Base Class (Template)

## Definition
```java
// Signature
public abstract class SingleParameterProvider implements ParameterProvider {
```

## Architecture & Concepts
The SingleParameterProvider is an abstract base class that serves as a foundational component within the server-side NPC Artificial Intelligence framework. Its primary architectural role is to enforce a strict, non-negotiable contract: a concrete implementation can, and must, only be responsible for a single, predefined parameter.

This class acts as a specialization of the more general ParameterProvider interface. By extending it, developers create a component that explicitly rejects any request for a parameter ID that does not match the one it was constructed with. This design pattern simplifies the logic for NPC sensors that are inherently tied to a single data point (e.g., distance to target, current light level), eliminating the need for more complex map-based lookups or conditional branching within the provider itself.

It functions as a guard and a validator within the AI's sensor configuration and data retrieval process. When the AI system queries for a provider, this class's implementation of getParameterProvider provides a definitive, binary answer: either "I am the correct provider for this parameter" or a hard failure via an exception, indicating a critical misconfiguration in the AI's setup.

### Lifecycle & Ownership
- **Creation:** Concrete subclasses are instantiated by higher-level AI configuration systems, typically during the assembly of an NPC's sensor suite. The specific integer parameter ID is a mandatory dependency injected via the constructor at this time.
- **Scope:** The object's lifetime is tightly coupled to its parent sensor or the NPC entity it belongs to. It persists as long as the sensor is active.
- **Destruction:** The object is managed by the Java Garbage Collector. It is marked for cleanup when the parent NPC entity or its sensor configuration is destroyed, with no explicit deallocation methods required.

## Internal State & Concurrency
- **State:** The base class is **immutable**. Its internal state consists of a single `private final int` field, which is initialized once at construction and never modified. Concrete subclasses may introduce mutable state, but the core validation logic of the base class remains stateless.
- **Thread Safety:** This base class is inherently **thread-safe**. The getParameterProvider method performs a simple, non-blocking read operation on a final field.
    - **Warning:** Thread safety is not guaranteed for subclasses. Implementers are responsible for ensuring that any additional state or logic introduced in a concrete provider is managed in a thread-safe manner, as NPC behaviors may be evaluated across multiple server threads.

## API Surface
The public contract is minimal, focusing exclusively on parameter validation.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getParameterProvider(int parameter) | ParameterProvider | O(1) | Validates if the input parameter matches the provider's configured parameter. Throws IllegalStateException on mismatch. |

## Integration Patterns

### Standard Usage
This is an abstract class and cannot be instantiated directly. The standard pattern is to extend it to create a provider for a specific, named parameter.

```java
// 1. Define a concrete provider for a specific sensor parameter.
// The parameter ID (e.g., 42) would typically be a shared constant.
public class HostilityLevelProvider extends SingleParameterProvider {

    public HostilityLevelProvider() {
        super(42); // Binds this provider exclusively to parameter 42
    }

    // Other methods from the ParameterProvider interface would be implemented here.
}

// 2. In the AI system, the provider is used for validation.
ParameterProvider provider = new HostilityLevelProvider();

try {
    // This call succeeds and returns the provider itself.
    ParameterProvider validatedProvider = provider.getParameterProvider(42);
    // ...proceed with using the validatedProvider

} catch (IllegalStateException e) {
    // This block is only reached if there is a severe configuration error.
    // For example, if provider.getParameterProvider(99) was called.
    log.error("AI configuration mismatch for HostilityLevelProvider!");
}
```

### Anti-Patterns (Do NOT do this)
- **Control Flow via Exceptions:** Do not use a try-catch block around getParameterProvider as a method of selecting the correct provider. The IllegalStateException signifies a catastrophic configuration error, not a recoverable runtime condition. The logic to select the correct provider must exist at a higher level, *before* this validation method is called.
- **Dynamic Parameter Assignment:** Do not attempt to modify the parameter ID after construction using reflection or other means. The entire purpose of this class is to enforce immutability for its assigned parameter.

## Data Pipeline
This class is not part of a continuous data stream. Instead, it acts as a gatekeeper during the AI's configuration and query phase.

> Flow:
> AI Sensor System needs data for `PARAM_ID` -> Queries a registry for a `ParameterProvider` -> Retrieves a candidate (e.g., `HostilityLevelProvider`) -> **SingleParameterProvider.getParameterProvider(PARAM_ID)** -> Throws `IllegalStateException` OR Returns `this` -> AI System uses the validated provider to fetch sensor data.

